# Auto installation

this script should update in future

```bash
#!/usr/bin/env bash
set -euo pipefail

########## EDIT ME ##########
# Cluster basics
CLUSTER_NAME="ceph"
PUBLIC_CIDR="10.0.0.0/24"
CLUSTER_CIDR="10.1.0.0/24"           # set same as PUBLIC_CIDR if single network
MON_IP="10.0.0.10"                   # this node’s public IP
SSH_USER="root"                      # we use root for automation
CEPH_RELEASE="18.2.0"

# Hosts: "hostname:ip:labels"
HOSTS=(
  "master:10.0.0.10:_admin"
  "worker1:10.0.0.11:"
  "worker2:10.0.0.12:"
)

## daemon cout ##
# Cluster service counts
MON_COUNT=3
MGR_COUNT=2
ALERTMANAGER_COUNT=1
GRAFANA_COUNT=1

# Host patterns
CRASH_PATTERN="*"
NODE_EXPORTER_PATTERN="*"

# OSD_LAYOUT=(
#   "master:/dev/sdb /dev/nvme0n1"
#   "worker1:/dev/sdb"
#   "worker2:/dev/sdb"
# )

OSD_LAYOUT=(
  "worker1:/dev/mapper/ubuntu--vg-lv--disk1 /dev/mapper/ubuntu--vg-lv--disk2 /dev/mapper/ubuntu--vg-lv--disk3"
  "worker2:/dev/mapper/ubuntu--vg-lv--disk1 /dev/mapper/ubuntu--vg-lv--disk2 /dev/mapper/ubuntu--vg-lv--disk3"
)


# Root SSH key path & comment
SSH_KEY_FILE="/root/.ssh/id_ed25519"
SSH_KEY_COMMENT="cephadm-root-$(date +%F)"
########## STOP EDIT ########

[ "$EUID" -eq 0 ] || { echo "Run this script as root."; exit 1; }

red()   { printf "\033[31m%s\033[0m\n" "$*"; }
green() { printf "\033[32m%s\033[0m\n" "$*"; }
info()  { printf "\033[36m%s\033[0m\n" "$*"; }

need_cmd() { command -v "$1" >/dev/null || { red "Missing $1"; exit 1; }; }
need_cmd ssh
need_cmd scp
need_cmd curl
need_cmd ssh-copy-id
need_cmd openssl

###########################################################################################
# 0) Create root SSH key and distribute to all nodes (first-time trust)
info "[0/8] Ensuring root SSH key exists and pushing it to all nodes…"

# Generate key if missing
if [ ! -f "$SSH_KEY_FILE" ]; then
  info "Generating SSH key: $SSH_KEY_FILE"
  mkdir -p /root/.ssh && chmod 700 /root/.ssh
  ssh-keygen -t ed25519 -a 100 -f "$SSH_KEY_FILE" -C "$SSH_KEY_COMMENT"
fi

info "Checking SSH prerequisites…"
red "Ensure ALL servers have these in /etc/ssh/sshd_config, then restart sshd:"
red "  PermitRootLogin yes"
red "  PubkeyAuthentication yes"
red "  PasswordAuthentication yes   # needed initially so ssh-copy-id can work"
echo


# Push key (will prompt for root password once per host, only first time)
for entry in "${HOSTS[@]}"; do
  ip="${entry#*:}"; ip="${ip%%:*}"
  host="${entry%%:*}"
  info "Installing key on ${host} (${ip})…"
  ssh-copy-id -o StrictHostKeyChecking=accept-new "${SSH_USER}@${ip}" || {
    red "ssh-copy-id failed for ${ip}. Check network/ssh/credentials and retry."
    exit 1
  }
done

# Verify passwordless works (key-based)
for entry in "${HOSTS[@]}"; do
  ip="${entry#*:}"; ip="${ip%%:*}"
  host="${entry%%:*}"
  if ssh -o BatchMode=yes root@"${ip}" "echo ok" >/dev/null 2>&1; then
    green "✔ Passwordless SSH to ${host} (${ip}) as root works"
  else
    red "✘ Passwordless SSH to ${host} (${ip}) as root FAILED"
    red "  On ALL servers, set in /etc/ssh/sshd_config and restart sshd:"
    red "    PermitRootLogin yes"
    red "    PubkeyAuthentication yes"
    red "    PasswordAuthentication yes   # keep enabled at least for the first run"
    red "  Then re-run this script."
    exit 1
  fi
done

###########################################################################################
# 1) Minimal node prep (swap off, chrony, lvm2, jq, podman) — with verbose, per-host output
info "[1/8] Prepping nodes (swap off, chrony, lvm2, jq, podman)…"

run_node_prep() {
  local host ip
  host="$1"; ip="$2"

  info "[prep][$host] starting"
  # We run a heredoc *through ssh bash -s* so stdout/stderr stream back.
  # sed prefixes each line with [host] to make logs readable.
  ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new "${SSH_USER}@${ip}" bash -s <<'REMOTE' 2>&1 | sed -u "s/^/[prep][$host] /"
set -euxo pipefail

if command -v apt >/dev/null; then
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -y
  apt-get install -y chrony lvm2 jq curl podman
elif command -v dnf >/dev/null; then
  dnf install -y chrony lvm2 jq curl podman
elif command -v yum >/dev/null; then
  yum install -y chrony lvm2 jq curl podman
else
  echo "No supported package manager found (need apt, dnf, or yum)"; exit 1
fi

systemctl enable --now chronyd || systemctl enable --now chrony || true
swapoff -a || true
sed -i.bak '/\sswap\s/d' /etc/fstab || true

echo "Node prep complete"
REMOTE
}

# Call it for each host
for entry in "${HOSTS[@]}"; do
  host="${entry%%:*}"
  ip="${entry#*:}"; ip="${ip%%:*}"
  run_node_prep "$host" "$ip"
done

###########################################################################################
# 1b) Verify packages and chrony on all nodes (fail fast if anything missing)
info "[2/8] Verifying required packages and Chrony status…"

REQUIRED_PKGS=(chrony lvm2 jq curl podman)
overall_ok=true

verify_host() {
  local entry="$1"
  local host="${entry%%:*}"
  local rest="${entry#*:}"
  local ip="${rest%%:*}"
  local osfam chrony_unit host_ok=true

  info "Checking ${host} (${ip})…"

  # Detect OS family & correct chrony unit name
  if ssh -o BatchMode=yes "${SSH_USER}@${ip}" 'command -v dpkg >/dev/null'; then
    osfam="debian"; chrony_unit="chrony"
  elif ssh -o BatchMode=yes "${SSH_USER}@${ip}" 'command -v rpm >/dev/null'; then
    osfam="rhel"; chrony_unit="chronyd"
  else
    red "  ✘ Unknown OS family (no dpkg or rpm)"; overall_ok=false; return
  fi

  # Package checks
  for pkg in "${REQUIRED_PKGS[@]}"; do
    if [ "$osfam" = "debian" ]; then
      if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "dpkg -s '$pkg' >/dev/null 2>&1"; then
        green "  ✔ $pkg installed"
      else
        red   "  ✘ $pkg NOT installed"
        host_ok=false
      fi
    else
      if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "rpm -q '$pkg' >/dev/null 2>&1"; then
        green "  ✔ $pkg installed"
      else
        red   "  ✘ $pkg NOT installed"
        host_ok=false
      fi
    fi
  done

  # Chrony service checks
  if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "systemctl is-active --quiet ${chrony_unit}"; then
    green "  ✔ ${chrony_unit} service is active"
  else
    red   "  ✘ ${chrony_unit} service is NOT active"
    host_ok=false
  fi
  if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "systemctl is-enabled --quiet ${chrony_unit}"; then
    green "  ✔ ${chrony_unit} service is enabled"
  else
    red   "  ✘ ${chrony_unit} service is NOT enabled"
    host_ok=false
  fi

  if $host_ok; then
    green "[OK] ${host} passed verification"
  else
    red   "[FAIL] ${host} did not meet prerequisites"
    overall_ok=false
  fi
}

for entry in "${HOSTS[@]}"; do
  verify_host "$entry"
done

if ! $overall_ok; then
  red "Prerequisite verification FAILED on one or more hosts. Fix the issues and re-run."
  exit 1
fi

green "All hosts passed prerequisite verification."

###########################################################################################
# 3) Install cephadm on bootstrap node (this host)
info "[3/8] Installing cephadm on bootstrap node…"

if ! command -v cephadm >/dev/null; then
  info "Downloading cephadm ${CEPH_RELEASE}…"
  curl --silent --remote-name --location \
    "https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm"

  chmod +x cephadm
  info "Adding Ceph repo for Reef release…"
  ./cephadm add-repo --release reef

  info "Installing cephadm…"
  ./cephadm install
else
  green "cephadm already installed, skipping download."
fi

# Verify cephadm is available
if which cephadm >/dev/null 2>&1; then
  green "✔ cephadm installation verified: $(which cephadm)"
else
  red "✘ cephadm not found after installation."
  exit 1
fi

###########################################################################################
# 4) Bootstrap cluster
info "[4/8] Bootstrapping cluster ${CLUSTER_NAME}…"
cephadm bootstrap \
  --cluster-network "${CLUSTER_CIDR}" \
  --mon-ip "${MON_IP}" \
  --initial-dashboard-user admin \
  --initial-dashboard-password "$(openssl rand -base64 12)" \
  --skip-monitoring-stack || true  # we'll place services via spec

###########################################################################################
# 5) Distribute cephadm orchestrator SSH key to other nodes (via 'ceph cephadm')
info "[5/8] Distributing cephadm SSH key to other nodes…"

# Ensure ceph-common is installed (for the 'ceph' CLI)
if command -v apt >/dev/null; then
  apt-get install -y ceph-common
elif command -v dnf >/dev/null; then
  dnf install -y ceph-common
elif command -v yum >/dev/null; then
  yum install -y ceph-common
fi

# Retrieve orchestrator pubkey
if PUBKEY="$(ceph cephadm get-pub-key 2>/dev/null)"; then
  :
else
  # Fallback if mgr not ready
  PUBKEY="$(cephadm get-pub-key 2>/dev/null || cat /etc/ceph/ceph.pub 2>/dev/null || true)"
fi

if [ -z "${PUBKEY:-}" ]; then
  red "Failed to obtain cephadm orchestrator SSH public key."
  red "Check 'ceph -s' and try again."
  exit 1
fi

# Add key on each host with inline comment (idempotent)
for entry in "${HOSTS[@]}"; do
  ip="${entry#*:}"; ip="${ip%%:*}"
  host="${entry%%:*}"

  KEY_WITH_COMMENT="$(printf "%s # ceph_pub\n" "$PUBKEY")"

  ssh "${SSH_USER}@${ip}" "
    set -e
    mkdir -p /root/.ssh && chmod 700 /root/.ssh
    touch /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/authorized_keys
    grep -Fq \"${PUBKEY}\" /root/.ssh/authorized_keys || printf '%s' '${KEY_WITH_COMMENT}' >> /root/.ssh/authorized_keys
    command -v restorecon >/dev/null && restorecon -Rv /root/.ssh || true
  " && green "✔ Installed cephadm pubkey on ${host} (${ip})" || {
    red "✘ Failed to install cephadm pubkey on ${host} (${ip})"
    exit 1
  }
done

###########################################################################################
# 6) Add hosts to the orchestrator WITH labels (idempotent) + error checks
info "[6/8] Adding hosts to Ceph orchestrator (with labels)…"

for entry in "${HOSTS[@]}"; do
  host="${entry%%:*}"
  rest="${entry#*:}"
  ip="${rest%%:*}"
  labels="${entry##*:}"

  # Build the add command
  if [ -n "${labels}" ]; then
    ADD_CMD=(ceph orch host add "${host}" "${ip}" --labels "${labels}")
  else
    ADD_CMD=(ceph orch host add "${host}" "${ip}")
  fi

  # Try to add with labels
  if ! out="$("${ADD_CMD[@]}" 2>&1)"; then
    if [[ "$out" == *"already exists"* ]]; then
      info "Host '${host}' already exists; ensuring labels…"
      # Ensure labels are present if provided
      if [ -n "${labels}" ]; then
        IFS=',' read -ra lbls <<< "${labels}"
        for lbl in "${lbls[@]}"; do
          [ -z "$lbl" ] && continue
          if ! ceph orch host label add "${host}" "${lbl}" >/dev/null 2>&1; then
            red "✘ Failed to add label '${lbl}' to host '${host}'"
            exit 1
          fi
        done
      fi
      green "✔ Host '${host}' is present and labeled"
    else
      red "✘ Failed to add host '${host}' (${ip}):"
      red "  $out"
      exit 1
    fi
  else
    green "✔ Added host '${host}' (${ip}) with labels '${labels}'"
  fi
done

# Show orchestrator host table
info "Current orchestrator hosts:"
if ! ceph orch host ls; then
  red "✘ 'ceph orch host ls' failed"
  exit 1
fi

###########################################################################################
# 7) Generate and apply cluster spec (core daemons only; NO OSD/MDS/RGW)
info "[7/8] Writing cluster spec (core services only)…"
cat > /tmp/cluster-spec.yaml <<EOF
---
service_type: mon
placement:
  count: ${MON_COUNT}

---
service_type: mgr
placement:
  count: ${MGR_COUNT}

---
service_type: crash
placement:
  host_pattern: "${CRASH_PATTERN}"

---
service_type: node-exporter
placement:
  host_pattern: "${NODE_EXPORTER_PATTERN}"

---
service_type: alertmanager
placement:
  count: ${ALERTMANAGER_COUNT}

---
service_type: grafana
placement:
  count: ${GRAFANA_COUNT}
EOF

# Apply and verify
if ceph orch apply -i /tmp/cluster-spec.yaml; then
  green "✔ Core service spec applied (no OSD/MDS/RGW)"
else
  red "✘ Failed to apply core service spec"
  exit 1
fi

info "Currently defined services:"
ceph orch ls || { red "✘ 'ceph orch ls' failed"; exit 1; }

###########################################################################################

# 8) Add OSDs via `ceph orch daemon add osd host:/dev/...`
info "[8/8] Creating OSDs using ceph orch daemon add…"

# Example OSD_LAYOUT (defined earlier in your script)
# OSD_LAYOUT=(
#   "worker1:/dev/mapper/ubuntu--vg-lv--disk1 /dev/mapper/ubuntu--vg-lv--disk2 /dev/mapper/ubuntu--vg-lv--disk3"
#   "worker2:/dev/mapper/ubuntu--vg-lv--disk1 /dev/mapper/ubuntu--vg-lv--disk2 /dev/mapper/ubuntu--vg-lv--disk3"
# )

any_added=false

# Helper: check if this host:device already has an OSD scheduled/running
osd_exists_for_device() {
  local host="$1" dev="$2"
  # Normalize device (readlink -f) on the target, then search in orch ps
  local ip=""
  for h in "${HOSTS[@]}"; do
    h_host="${h%%:*}"; h_rest="${h#*:}"; h_ip="${h_rest%%:*}"
    [[ "$h_host" == "$host" ]] && { ip="$h_ip"; break; }
  done
  [[ -z "$ip" ]] && { red "  ↳ host '$host' not found in HOSTS"; return 1; }

  # Resolve the real path (e.g., mapper/by-id -> /dev/sdX) on the remote host
  realpath=$(ssh -o BatchMode=yes "${SSH_USER}@${ip}" "readlink -f '$dev' 2>/dev/null" || true)
  [[ -z "$realpath" ]] && realpath="$dev"

  # Look for an existing daemon referencing that device
  ceph orch ps --daemon_type osd -f json 2>/dev/null | jq -e --arg host "$host" --arg dev "$realpath" '
    .[] | select(.hostname == $host) | ( .extra_info // "" ) + " " + ( .container_image_id // "" )
  ' >/dev/null 2>&1 && false # (don’t rely on extra_info reliably across versions)

  # Fallback: grep ps text for host and device path
  if ceph orch ps --daemon_type osd 2>/dev/null | grep -F "$host" | grep -F "$realpath" >/dev/null 2>&1; then
    return 0
  fi
  return 1
}

for entry in "${OSD_LAYOUT[@]}"; do
  host="${entry%%:*}"
  devs_raw="${entry#*:}"

  # Find host IP (for remote checks)
  ip=""
  for h in "${HOSTS[@]}"; do
    h_host="${h%%:*}"; h_rest="${h#*:}"; h_ip="${h_rest%%:*}"
    if [[ "$h_host" == "$host" ]]; then ip="$h_ip"; break; fi
  done
  if [[ -z "$ip" ]]; then
    red "✘ OSD host '${host}' not found in HOSTS array. Fix OSD_LAYOUT or HOSTS."
    exit 1
  fi

  # Split device list
  read -r -a DEVICES <<< "$devs_raw"
  if [[ "${#DEVICES[@]}" -eq 0 ]]; then
    info "[osd][$host] No devices listed; skipping."
    continue
  fi

  for d in "${DEVICES[@]}"; do
    info "[osd][$host] validating $d …"
    # Validate on remote: block device and not mounted
    if ! ssh -o BatchMode=yes "${SSH_USER}@${ip}" bash -lc "
        set -e
        test -b '$d'
        # Ensure no mountpoints on the device or its partitions
        lsblk -no TYPE,MOUNTPOINT '$d' | awk '
          \$1==\"disk\" && NF==1 { next }      # disk line with no mountpoint OK
          \$1==\"disk\" && NF>1  { exit 1 }    # disk line with mountpoint -> BAD
          \$1==\"part\" && NF>1  { exit 1 }    # any partition mounted -> BAD
          END { if (NR==0) exit 1 }            # no lsblk output? BAD
        '
      " >/dev/null 2>&1; then
      red "  ✘ Device $d on ${host} is not suitable (missing or mounted)."
      exit 1
    fi

    # Skip if already added
    if osd_exists_for_device "$host" "$d"; then
      info "  ↳ OSD for ${host}:${d} appears to exist; skipping."
      continue
    fi

    # Add OSD
    info "  → ceph orch daemon add osd ${host}:${d}"
    if ceph orch daemon add osd "${host}:${d}"; then
      green "  ✔ queued OSD on ${host}:${d}"
      any_added=true
    else
      red "  ✘ failed to add OSD on ${host}:${d}"
      exit 1
    fi
  done
done

# Show result
info "Current OSD daemons:"
ceph orch ps --daemon_type osd || true
$any_added && green "✔ OSD add operations submitted." || info "No new OSDs were added (all present or none configured)."





# 9) Final checks
info "[8/8] Cluster coming up. Useful commands:"
echo "  ceph -s"
echo "  ceph orch ps"
echo "  ceph fs status"

green "Done! Give daemons a few minutes to settle."

```
