# Auto installation

this script should update in future

```bash
#!/usr/bin/env bash
set -euo pipefail

########## EDIT ME ##########
# Cluster basics
CLUSTER_NAME="ceph"
PUBLIC_CIDR="192.168.244.0/24"
CLUSTER_CIDR="192.168.244.0/24"           # set same as PUBLIC_CIDR if single network
MON_IP="192.168.244.8"                    # this node’s public IP
SSH_USER="root"                           # we use root for automation
CEPH_RELEASE="18.2.0"

# Hosts: "hostname:ip:labels"
HOSTS=(
  "stage-master:192.168.244.8:_admin,mon,mgr"
  "stage-worker1:192.168.244.47:mon,osd"
  "stage-worker2:192.168.244.10:mon,osd"
  "stage-worker3:192.168.244.15:osd"
)

# Cluster service counts
MON_COUNT=3
MGR_COUNT=1
ALERTMANAGER_COUNT=1
GRAFANA_COUNT=1

# Host patterns
CRASH_PATTERN="*"
NODE_EXPORTER_PATTERN="*"

# Root SSH key path & comment
SSH_KEY_FILE="/root/.ssh/id_ed25519"
SSH_KEY_COMMENT="cephadm-root-$(date +%F)"
########## STOP EDIT ########

[ "$EUID" -eq 0 ] || { echo "Run this script as root."; exit 1; }

red()   { printf "\033[31m%s\033[0m\n" "$*"; }
green() { printf "\033[32m%s\033[0m\n" "$*"; }
info()  { printf "\033[36m%s\033[0m\n" "$*"; }
bold()  { printf "\033[1m%s\033[0m\n" "$*"; }
yellow() { echo -e "\e[33m$*\e[0m"; }
blue()   { echo -e "\e[34m$*\e[0m"; }
warn()   { yellow "[WARN] $*"; }
error()  { red "[ERROR] $*"; }

need_cmd() { command -v "$1" >/dev/null || { red "Missing $1"; exit 1; }; }
need_cmd ssh
need_cmd scp
need_cmd curl
need_cmd ssh-copy-id
need_cmd openssl
need_cmd jq

# ----------------- Function: Initialization -----------------
initialize_cluster() {
###########################################################################################
# 0) Create root SSH key and distribute to all nodes (first-time trust)
info "[0/7] Ensuring root SSH key exists and pushing it to all nodes…"

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

for entry in "${HOSTS[@]}"; do
  ip="${entry#*:}"; ip="${ip%%:*}"
  host="${entry%%:*}"
  info "Installing key on ${host} (${ip})…"
  ssh-copy-id -o StrictHostKeyChecking=accept-new "${SSH_USER}@${ip}" || {
    red "ssh-copy-id failed for ${ip}. Check network/ssh/credentials and retry."
    exit 1
  }
done

for entry in "${HOSTS[@]}"; do
  ip="${entry#*:}"; ip="${ip%%:*}"
  host="${entry%%:*}"
  if ssh -o BatchMode=yes root@"${ip}" "echo ok" >/dev/null 2>&1; then
    green "✔ Passwordless SSH to ${host} (${ip}) as root works"
  else
    red "✘ Passwordless SSH to ${host} (${ip}) as root FAILED"
    exit 1
  fi
done

###########################################################################################
# 1) Minimal node prep
info "[1/7] Prepping nodes (swap off, chrony, lvm2, jq, podman)…"

run_node_prep() {
  local host ip
  host="$1"; ip="$2"
  info "[prep][$host] starting"
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

for entry in "${HOSTS[@]}"; do
  host="${entry%%:*}"
  ip="${entry#*:}"; ip="${ip%%:*}"
  run_node_prep "$host" "$ip"
done

###########################################################################################
# 2) Verify packages and chrony
info "[2/7] Verifying required packages and Chrony status…"
REQUIRED_PKGS=(chrony lvm2 jq curl podman)
overall_ok=true

verify_host() {
  local entry="$1"
  local host="${entry%%:*}"
  local rest="${entry#*:}"
  local ip="${rest%%:*}"
  local osfam chrony_unit host_ok=true
  info "Checking ${host} (${ip})…"
  if ssh -o BatchMode=yes "${SSH_USER}@${ip}" 'command -v dpkg >/dev/null'; then
    osfam="debian"; chrony_unit="chrony"
  elif ssh -o BatchMode=yes "${SSH_USER}@${ip}" 'command -v rpm >/dev/null'; then
    osfam="rhel"; chrony_unit="chronyd"
  else
    red "  ✘ Unknown OS family"; overall_ok=false; return
  fi
  for pkg in "${REQUIRED_PKGS[@]}"; do
    if [ "$osfam" = "debian" ]; then
      if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "dpkg -s '$pkg' >/dev/null 2>&1"; then
        green "  ✔ $pkg installed"
      else
        red   "  ✘ $pkg NOT installed"; host_ok=false
      fi
    else
      if ssh -o BatchMode=yes "${SSH_USER}@${ip}" "rpm -q '$pkg' >/dev/null 2>&1"; then
        green "  ✔ $pkg installed"
      else
        red   "  ✘ $pkg NOT installed"; host_ok=false
      fi
    fi
  done
  ssh -o BatchMode=yes "${SSH_USER}@${ip}" "systemctl is-active --quiet ${chrony_unit}" \
    && green "  ✔ ${chrony_unit} active" || { red "  ✘ ${chrony_unit} not active"; host_ok=false; }
  ssh -o BatchMode=yes "${SSH_USER}@${ip}" "systemctl is-enabled --quiet ${chrony_unit}" \
    && green "  ✔ ${chrony_unit} enabled" || { red "  ✘ ${chrony_unit} not enabled"; host_ok=false; }
  $host_ok && green "[OK] ${host} passed" || { red "[FAIL] ${host} prerequisites"; overall_ok=false; }
}

for entry in "${HOSTS[@]}"; do verify_host "$entry"; done
$overall_ok || { red "Prerequisite verification FAILED"; exit 1; }
green "All hosts passed verification."

###########################################################################################
# 3) Install cephadm
info "[3/7] Installing cephadm on bootstrap node…"
if ! command -v cephadm >/dev/null; then
  info "Downloading cephadm ${CEPH_RELEASE}…"
  curl --silent --remote-name --location \
    "https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm"
  chmod +x cephadm
  ./cephadm add-repo --release reef
  ./cephadm install
else
  green "cephadm already installed"
fi
command -v cephadm >/dev/null || { red "cephadm not found"; exit 1; }

###########################################################################################
# 4) Bootstrap cluster
info "[4/7] Bootstrapping cluster ${CLUSTER_NAME}…"
cephadm bootstrap \
  --cluster-network "${CLUSTER_CIDR}" \
  --mon-ip "${MON_IP}" \
  --initial-dashboard-user admin \
  --initial-dashboard-password "$(openssl rand -base64 12)" \
  --skip-monitoring-stack || true

###########################################################################################
# 5) Distribute cephadm orchestrator SSH key
info "[5/7] Distributing cephadm SSH key to other nodes…"
if command -v apt >/dev/null; then apt-get install -y ceph-common
elif command -v dnf >/dev/null; then dnf install -y ceph-common
elif command -v yum >/dev/null; then yum install -y ceph-common; fi

if PUBKEY="$(ceph cephadm get-pub-key 2>/dev/null)"; then :; else
  PUBKEY="$(cephadm get-pub-key 2>/dev/null || cat /etc/ceph/ceph.pub 2>/dev/null || true)"
fi
[ -n "${PUBKEY:-}" ] || { red "Failed to obtain cephadm key"; exit 1; }

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
  " && green "✔ Installed pubkey on ${host}" || { red "✘ Failed pubkey on ${host}"; exit 1; }
done

###########################################################################################
# 6) Add hosts with labels
info "[6/7] Adding hosts to Ceph orchestrator…"
for entry in "${HOSTS[@]}"; do
  host="${entry%%:*}"
  rest="${entry#*:}"
  ip="${rest%%:*}"
  labels="${entry##*:}"
  if [ -n "${labels}" ]; then ADD_CMD=(ceph orch host add "${host}" "${ip}" --labels "${labels}")
  else ADD_CMD=(ceph orch host add "${host}" "${ip}"); fi
  if ! out="$("${ADD_CMD[@]}" 2>&1)"; then
    if [[ "$out" == *"already exists"* ]]; then
      green "Host '${host}' already exists"
    else
      red "✘ Failed to add host '${host}' (${ip}): $out"; exit 1
    fi
  else
    green "✔ Added host '${host}' (${ip})"
  fi
done
ceph orch host ls

###########################################################################################
# 7) Apply core service spec
info "[7/7] Applying core service spec…"
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

ceph orch apply -i /tmp/cluster-spec.yaml
ceph orch ls
green "Initialization complete (no OSDs yet)."
}

# ----------------- Function: Show device inventory -----------------
# ---- helpers (add these) ----
ceph_ready() {
  # returns 0 if ceph CLI can reach a cluster and orch is responding
  ceph status -f json >/dev/null 2>&1 || return 1
  ceph orch host ls -f json >/dev/null 2>&1 || return 1
  return 0
}

host_in_cluster() {
  local host="$1"
  ceph orch host ls -f json 2>/dev/null \
    | jq -e --arg h "$host" '. // [] | map(.hostname) | index($h) != null' >/dev/null
}

# ---- replace your old show_host_devices_table with this safe version ----
show_host_devices_table() {
  local host="$1"
  ceph orch device ls --host "$host" -f json 2>/dev/null \
    | jq -r --arg host "$host" '
        .[] | select(.name==$host) | .devices[]? |
        [ .path, .available, ( .rejected_reasons | join("; ") ) ] | @tsv
      ' \
    | awk 'BEGIN {
             printf("\n%-15s %-10s %s\n","DEVICE","AVAILABLE","REASONS")
             print "----------------------------------------------"
           }
           { printf("%-15s %-10s %s\n",$1,$2,$3) }'
  echo
}

# ---- replace your old show_inventory with this version ----
show_inventory() {
  if ! ceph_ready; then
    red "Ceph is not initialized yet, or the orchestrator is not responding."
    info "Use menu option 1 (Initialize cluster) first, then try again."
    return 1
  fi

  for entry in "${HOSTS[@]}"; do
    local host="${entry%%:*}"
    info "Devices on ${host}:"
    show_host_devices_table "$host"
  done
}


# ----------------- Function: Add OSDs -----------------
add_osds() {
  if ! ceph_ready; then
    red "Ceph cluster is not ready. Please initialize first."
    return 1
  fi

  # Build menu of hosts with 'osd' label
  local map=() i=1
  for entry in "${HOSTS[@]}"; do
    local name="${entry%%:*}"
    local labels="${entry##*:}"
    if [[ "$labels" == *"osd"* ]]; then
      echo "  [$i] $name"
      map[$i]="$name"
      i=$((i+1))
    fi
  done
  (( ${#map[@]} > 0 )) || { red "No OSD hosts found in inventory"; return 1; }

  read -rp "Select host number: " choice
  local host="${map[$choice]:-}"
  [ -n "$host" ] || { red "Invalid choice"; return 1; }

  info "Available devices on $host:"
  show_host_devices_table "$host"

  read -rp "Enter devices to add (space separated, e.g. /dev/vdb /dev/vdc): " -a devs
  [ ${#devs[@]} -gt 0 ] || { info "No devices selected"; return 0; }

  for d in "${devs[@]}"; do
    info "Adding OSD: $host:$d"
    if ceph orch daemon add osd "$host:$d"; then
      green "✔ Added $d on $host"
    else
      red "✘ Failed to add $d on $host"
    fi
  done
}


# ----------------- Function: Ceph Information -----------------
ceph_info_menu() {
  if ! ceph_ready; then
    red "Ceph cluster is not ready. Please initialize first."
    return 1
  fi

  while true; do
    echo
    bold "Ceph Information Menu"
    echo "  1) Cluster status (ceph -s)"
    echo "  2) OSD tree (ceph osd tree)"
    echo "  3) Host list (ceph orch host ls)"
    echo "  4) Service list (ceph orch ls)"
    echo "  5) Daemon list (ceph orch ps)"
    echo "  6) FS status (ceph fs status)"
    echo "  7) Exit to main menu"
    read -rp "Choose an option: " choice

    case "$choice" in
      1) info "Cluster status:"; ceph -s ;;
      2) info "OSD tree:"; ceph osd tree ;;
      3) info "Hosts:"; ceph orch host ls ;;
      4) info "Services:"; ceph orch ls ;;
      5) info "Daemons:"; ceph orch ps ;;
      6) info "Filesystem status:"; ceph fs status ;;
      7) return 0 ;;
      *) red "Invalid choice";;
    esac
  done
}

# ----------------- Function: Check Chrony health (skew/offset) -----------------
check_chrony_health() {
  local THRESH_LOCAL_MS=500         # warn if abs(offset) > 500 ms
  local THRESH_CLUSTER_SKEW_MS=1000 # warn if (max-min) > 1000 ms

  printf "\n%-16s %-8s %-6s %-7s %-16s\n" "HOST" "UNIT" "SYNC" "STRAT" "LAST_OFFSET(ms)"
  printf "%-16s %-8s %-6s %-7s %-16s\n" "----------------" "--------" "------" "------" "----------------"

  local min_ms=999999999 max_ms=-999999999 any_warn=0

  for entry in "${HOSTS[@]}"; do
    local host="${entry%%:*}"
    local ip="${entry#*:}"; ip="${ip%%:*}"

    # Collect minimal, clean key=value pairs remotely
    local line
    line=$(ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new "${SSH_USER}@${ip}" '
      unit=none
      if systemctl is-active --quiet chrony 2>/dev/null;  then unit=chrony;  fi
      if systemctl is-active --quiet chronyd 2>/dev/null; then unit=chronyd; fi

      sync=$(timedatectl show -p NTPSynchronized --value 2>/dev/null || echo unknown)

      track=$(chronyc tracking 2>/dev/null || true)
      # Extract with sed (no nested quotes headaches)
      stratum=$(printf "%s\n" "$track" | sed -n "s/^Stratum[[:space:]]*:[[:space:]]*\\([0-9][0-9]*\\).*$/\\1/p" | head -n1)
      last_s=$(printf "%s\n" "$track" | sed -n "s/^Last offset[[:space:]]*:[[:space:]]*\\([-0-9.][0-9.-]*\\).*$/\\1/p" | head -n1)

      # Defaults if missing
      [ -z "$stratum" ] && stratum="?"
      [ -z "$last_s" ] && last_s="nan"

      echo "UNIT=$unit SYNC=$sync STRATUM=$stratum LAST_S=$last_s"
    ' 2>/dev/null) || {
      printf "%-16s %-8s %-6s %-7s %-16s\n" "$host" "unreach" "-" "-" "-"
      any_warn=1
      continue
    }

    # Parse key=value pairs reliably
    local unit sync stratum last_s
    unit=$(printf "%s\n" "$line"    | tr " " "\n" | sed -n 's/^UNIT=//p'    | head -n1)
    sync=$(printf "%s\n" "$line"    | tr " " "\n" | sed -n 's/^SYNC=//p'    | head -n1)
    stratum=$(printf "%s\n" "$line" | tr " " "\n" | sed -n 's/^STRATUM=//p' | head -n1)
    last_s=$(printf "%s\n" "$line"  | tr " " "\n" | sed -n 's/^LAST_S=//p'  | head -n1)

    # Convert seconds→ms (rounded) on the local side
    local last_ms="nan"
    if [[ "$last_s" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]; then
      last_ms=$(awk -v s="$last_s" 'BEGIN{printf("%.0f", s*1000)}')
      (( last_ms < min_ms )) && min_ms=$last_ms
      (( last_ms > max_ms )) && max_ms=$last_ms
    fi

    local sync_flag="-"
    [[ "$sync" =~ ^(yes|1|true)$ ]] && sync_flag="yes"

    printf "%-16s %-8s %-6s %-7s %-16s\n" "$host" "$unit" "$sync_flag" "$stratum" "$last_ms"

    # Per-host warnings
    if [[ "$unit" == "none" ]]; then
      red "[$host] chrony/chronyd NOT running."
      any_warn=1
    fi
    if ! [[ "$sync" =~ ^(yes|1|true)$ ]]; then
      red "[$host] System clock NOT synchronized (timedatectl NTPSynchronized=$sync)."
      any_warn=1
    fi
    if [[ "$last_ms" =~ ^-?[0-9]+$ ]]; then
      local abs=$(( last_ms<0 ? -last_ms : last_ms ))
      if (( abs > THRESH_LOCAL_MS )); then
        red "[$host] Offset ${last_ms} ms exceeds ${THRESH_LOCAL_MS} ms."
        any_warn=1
      fi
    fi
  done

  # Cluster skew (max - min)
  if [[ $min_ms -ne 999999999 && $max_ms -ne -999999999 ]]; then
    local skew=$(( max_ms - min_ms ))
    if (( skew > THRESH_CLUSTER_SKEW_MS )); then
      red "Cluster clock skew high: ~${skew} ms (max-min)."
      any_warn=1
    else
      green "Cluster clock skew ~${skew} ms."
    fi
  else
    info "Cluster skew could not be computed (missing offsets)."
  fi

  (( any_warn == 0 )) && green "Chrony health looks good on all hosts."
}


ntp_set_master_and_clients() {
  # Find master's hostname (optional, just for logging)
  local MASTER_HOST=""
  for entry in "${HOSTS[@]}"; do
    local h="${entry%%:*}"
    local ip="${entry#*:}"; ip="${ip%%:*}"
    if [ "$ip" = "$MON_IP" ]; then
      MASTER_HOST="$h"
      break
    fi
  done
  [ -z "$MASTER_HOST" ] && MASTER_HOST="$MON_IP"

  info "Configuring master ${MASTER_HOST} (${MON_IP}) as NTP server…"
  # NOTE: unquoted <<EOF so ${PUBLIC_CIDR} expands
  ssh -o StrictHostKeyChecking=accept-new "${SSH_USER}@${MON_IP}" "bash -s" <<EOF
set -euo pipefail
cat >/etc/chrony/chrony.conf <<EOC
# Chrony as LAN time server
local stratum 10
allow ${PUBLIC_CIDR}
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
EOC

# restart chrony (or chronyd)
systemctl restart chrony 2>/dev/null || systemctl restart chronyd 2>/dev/null
systemctl is-active --quiet chrony  || systemctl is-active --quiet chronyd  || (echo "chrony/chronyd not active"; exit 1)
chronyc tracking || true
EOF

  # Configure workers to use the master
  for entry in "${HOSTS[@]}"; do
    host="${entry%%:*}"
    ip="${entry#*:}"; ip="${ip%%:*}"
    [ "$ip" = "$MON_IP" ] && continue

    info "Configuring worker ${host} (${ip}) to sync from ${MON_IP}…"
    # NOTE: unquoted <<EOF so ${MON_IP} expands
    ssh -o StrictHostKeyChecking=accept-new "${SSH_USER}@${ip}" "bash -s" <<EOF
set -euo pipefail
cat >/etc/chrony/chrony.conf <<EOC
server ${MON_IP} iburst
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
EOC

systemctl restart chrony 2>/dev/null || systemctl restart chronyd 2>/dev/null
systemctl is-active --quiet chrony  || systemctl is-active --quiet chronyd  || (echo "chrony/chronyd not active"; exit 1)

# give it a nudge
chronyc burst 4/4 || true
chronyc makestep || true
chronyc sources -v || true
timedatectl show -p NTPSynchronized --value || true
EOF
  done
  green "NTP server/client config applied."
}

# ----------------- Function: CRUSH Map Management -----------------
crush_map() {
  if ! ceph_ready; then
    red "Ceph cluster is not ready. Please initialize first."
    return 1
  fi

  while true; do
    echo
    bold "CRUSH Map Management Menu"
    echo "  1) Show current CRUSH map (requires crushtool)"
    echo "  2) Show OSD tree (quick overview)"
    echo "  3) Create new CRUSH root"
    echo "  4) Create new host and add to specific root"
    echo "  5) Move existing host to different root"
    echo "  6) Move OSD to different host"
    echo "  7) Create CRUSH rule for specific root"
    echo "  8) List all CRUSH rules"
    echo "  9) Exit to main menu"
    read -rp "Choose an option: " choice

    case "$choice" in
      1)
        # Check for crushtool only when needed
        if ! command -v crushtool >/dev/null; then
          red "crushtool not found. Please ensure ceph-base is installed."
          info "Run the following to install ceph-base:"
          if command -v apt >/dev/null; then
            echo "  apt-get install -y ceph-base"
          elif command -v dnf >/dev/null; then
            echo "  dnf install -y ceph-base"
          elif command -v yum >/dev/null; then
            echo "  yum install -y ceph-base"
          else
            red "No supported package manager found (need apt, dnf, or yum)."
          fi
          continue
        fi
        
        info "Retrieving and displaying current CRUSH map..."
        if ceph osd getcrushmap -o /tmp/crushmap.bin && \
           crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt; then
          cat /tmp/crushmap.txt
          green "CRUSH map displayed successfully."
        else
          red "Failed to retrieve or display CRUSH map. Ensure the cluster is healthy and try again."
        fi
        ;;
      2)
        info "Current OSD tree:"
        ceph osd tree
        ;;
      3)
        read -rp "Enter name for new CRUSH root: " root_name
        if [ -z "$root_name" ]; then
          red "Root name cannot be empty!"
          continue
        fi
        
        # Check if root already exists
        if ceph osd crush tree | grep -q "root $root_name"; then
          red "Root '$root_name' already exists!"
          continue
        fi
        
        info "Creating new CRUSH root: $root_name"
        if ceph osd crush add-bucket "$root_name" root; then
          green "Root '$root_name' created successfully."
          info "You can now add hosts to this root using option 4."
        else
          red "Failed to create root '$root_name'."
        fi
        ;;
      4)
        info "Available roots:"
        ceph osd tree | grep "root" | awk '{print $4}'
        echo
        
        read -rp "Enter new host name: " host_name
        read -rp "Enter root name to add host to: " root_name
        
        if [ -z "$host_name" ] || [ -z "$root_name" ]; then
          red "Host name and root name cannot be empty!"
          continue
        fi
        
        # Check if host already exists
        if ceph osd crush tree | grep -q "host $host_name"; then
          red "Host '$host_name' already exists!"
          continue
        fi
        
        # Verify root exists
        if ! ceph osd crush tree | grep -q "root $root_name"; then
          red "Root '$root_name' does not exist! Create it first using option 3."
          continue
        fi
        
        info "Creating new host '$host_name' and adding to root '$root_name'"
        if ceph osd crush add-bucket "$host_name" host && \
           ceph osd crush move "$host_name" root="$root_name"; then
          green "Host '$host_name' created and added to root '$root_name' successfully."
          info "You can now add OSDs to this host using option 6."
        else
          red "Failed to create host '$host_name' or add it to root '$root_name'."
        fi
        ;;
      5)
        info "Available hosts:"
        ceph osd tree | grep "host" | grep -v "root" | awk '{print $4}'
        info "Available roots:"
        ceph osd tree | grep "root" | awk '{print $4}'
        
        read -rp "Enter host name to move: " host_name
        read -rp "Enter destination root name: " root_name
        
        if [ -z "$host_name" ] || [ -z "$root_name" ]; then
          red "Host name and root name cannot be empty!"
          continue
        fi
        
        # Verify host exists
        if ! ceph osd crush tree | grep -q "host $host_name"; then
          red "Host '$host_name' does not exist!"
          continue
        fi
        
        # Verify root exists
        if ! ceph osd crush tree | grep -q "root $root_name"; then
          red "Root '$root_name' does not exist! Create it first using option 3."
          continue
        fi
        
        info "Moving host '$host_name' to root '$root_name'"
        if ceph osd crush move "$host_name" root="$root_name"; then
          green "Host '$host_name' moved to root '$root_name' successfully."
        else
          red "Failed to move host '$host_name' to root '$root_name'."
        fi
        ;;
      6)
        info "Available OSDs:"
        ceph osd tree
        echo
        info "Available hosts:"
        ceph osd tree | grep "host" | grep -v "root" | awk '{print $4}'
        
        read -rp "Enter OSD ID to move (e.g., 0, 1, 2): " osd_id
        read -rp "Enter destination host name: " host_name
        
        if [ -z "$osd_id" ] || [ -z "$host_name" ]; then
          red "OSD ID and host name cannot be empty!"
          continue
        fi
        
        # Verify OSD exists
        if ! ceph osd tree | grep -q "osd.$osd_id "; then
          red "OSD.$osd_id does not exist!"
          continue
        fi
        
        # Verify host exists
        if ! ceph osd crush tree | grep -q "host $host_name"; then
          red "Host '$host_name' does not exist! Create it first using option 4."
          continue
        fi
        
        # Get OSD weight
        osd_weight=$(ceph osd tree | grep "osd.$osd_id " | awk '{print $3}')
        
        info "Moving OSD.$osd_id (weight: $osd_weight) to host '$host_name'"
        if ceph osd crush set "osd.$osd_id" "$osd_weight" host="$host_name"; then
          green "OSD.$osd_id moved to host '$host_name' successfully."
        else
          red "Failed to move OSD.$osd_id to host '$host_name'."
        fi
        ;;
      7)
        info "Available roots:"
        ceph osd tree | grep "root" | awk '{print $4}'
        echo
        info "Current CRUSH rules:"
        ceph osd crush rule ls
        echo
        
        # Explanation of failure domains
        bold "=== Understanding Failure Domains ==="
        echo "The failure domain determines how Ceph replicates data for fault tolerance:"
        echo
        echo "  host     - Replicas spread across different hosts (most common)"
        echo "  rack     - Replicas spread across different racks"
        echo "  row      - Replicas spread across different rows"
        echo "  datacenter - Replicas spread across different data centers"
        echo "  zone     - Replicas spread across different zones/availability zones"
        echo "  region   - Replicas spread across different regions"
        echo
        echo "For example:"
        echo "  - If you choose 'host', Ceph will place replicas on OSDs from different hosts"
        echo "  - If you choose 'rack', Ceph will place replicas on OSDs from different racks"
        echo "  - This ensures data survives failures at the chosen level"
        echo
        echo "Recommendation:"
        echo "  - For single rack: use 'host'"
        echo "  - For multiple racks: use 'rack'"
        echo "  - For multiple data centers: use 'datacenter' or 'zone'"
        echo
        
        read -rp "Enter name for new CRUSH rule: " rule_name
        read -rp "Enter root name for this rule: " root_name
        read -rp "Enter failure domain type (host, rack, datacenter, etc.): " failure_domain
        
        if [ -z "$rule_name" ] || [ -z "$root_name" ] || [ -z "$failure_domain" ]; then
          red "Rule name, root name, and failure domain cannot be empty!"
          continue
        fi
        
        # Check if rule already exists
        if ceph osd crush rule ls | grep -q "^$rule_name$"; then
          red "CRUSH rule '$rule_name' already exists!"
          continue
        fi
        
        # Verify root exists
        if ! ceph osd crush tree | grep -q "root $root_name"; then
          red "Root '$root_name' does not exist! Create it first using option 3."
          continue
        fi
        
        info "Creating CRUSH rule '$rule_name' for root '$root_name' with failure domain '$failure_domain'"
        if ceph osd crush rule create-replicated "$rule_name" "$root_name" "$failure_domain"; then
          green "CRUSH rule '$rule_name' created successfully."
          echo
          info "You can now use this rule when creating pools:"
          echo "  ceph osd pool create <pool_name> <pg_num> <pgp_num> replicated $rule_name"
          echo
          info "Example:"
          echo "  ceph osd pool create my_pool 32 32 replicated $rule_name"
        else
          red "Failed to create CRUSH rule '$rule_name'."
        fi
        ;;
      8)
        info "Current CRUSH rules:"
        ceph osd crush rule ls
        echo
        info "Rule details:"
        for rule in $(ceph osd crush rule ls); do
          echo "=== Rule: $rule ==="
          ceph osd crush rule dump "$rule"
          echo
        done
        ;;
      9)
        return 0
        ;;
      *)
        red "Invalid choice"
        ;;
    esac
  done
}

create_pool_with_rule() {
  if ! ceph_ready; then
    red "Ceph cluster is not ready. Please initialize first."
    read -rp "Press Enter to continue..."
    return 1
  fi

  echo
  bold "Create Pool with CRUSH Rule"
  echo "============================"
  info "Available CRUSH rules:"
  ceph osd crush rule ls || { red "Could not list CRUSH rules"; return 1; }
  echo

  read -rp "Enter pool name: " pool_name
  read -rp "Enter number of placement groups (PGs) [blank for autoscaler]: " pg_num
  read -rp "Enter CRUSH rule name: " rule_name
  read -rp "Enter replica size (default: 2): " size
  read -rp "Enter minimum replica size (default: 1): " min_size

  size=${size:-2}
  min_size=${min_size:-1}

  if [ -z "$pool_name" ] || [ -z "$rule_name" ]; then
    red "Pool name and CRUSH rule name are required!"
    read -rp "Press Enter to continue..."
    return 1
  fi

  # Validate rule exists
  if ! ceph osd crush rule ls | awk '{print $1}' | grep -qx "$rule_name"; then
    red "CRUSH rule '$rule_name' does not exist!"
    info "Available rules:"
    ceph osd crush rule ls
    read -rp "Press Enter to continue..."
    return 1
  fi
  echo "DEBUG: rule exist"

  # Determine rule root and failure domain type (prefers jq)
  if command -v jq >/dev/null 2>&1; then
    rule_root=$(ceph osd crush rule dump "$rule_name" -f json | jq -r '.steps[] | select(.op=="take") | .item_name' | head -1)
    failure_type=$(ceph osd crush rule dump "$rule_name" -f json | jq -r '.steps[] | select(.op|test("^chooseleaf")) | .type // "host"' | head -1)
  else
    rule_root=$(ceph osd crush rule dump "$rule_name" | awk -F'"' '/"op": "take"/{getline; if ($2=="item_name") print $4;}' | head -1)
    failure_type="host"
  fi

  if [ -z "$rule_root" ]; then
    red "Could not determine root bucket for rule '$rule_name'. Install 'jq' for reliable parsing."
    read -rp "Press Enter to continue..."
    return 1
  fi

  # Count available distinct failure domains (hosts by default) under the root
  available_domains="unknown"
  if command -v jq >/dev/null 2>&1; then
    available_domains=$(ceph osd crush tree -f json | \
      jq --arg root "$rule_root" --arg t "${failure_type:-host}" '
        .nodes as $nodes
        | ($nodes[] | select(.name==$root)) as $rootNode
        | [ $rootNode | .. | objects | select(.type?==$t) | .name ] | unique | length
      ')
  fi

  if [ "$available_domains" != "unknown" ] && [ "$available_domains" -gt 0 ] && [ "$size" -gt "$available_domains" ]; then
    red "Warning: replica size $size > available $failure_type(s) ($available_domains) under root '$rule_root'."
    read -rp "Continue anyway? (y/N): " proceed
    if [[ "$proceed" != "y" && "$proceed" != "Y" ]]; then
      read -rp "Press Enter to continue..."
      return 1
    fi
  fi
  echo "DEBUG: replica size"

  echo
  bold "Configuration Summary:"
  echo "Pool name: $pool_name"
  if [ -n "$pg_num" ]; then
    echo "PGs: $pg_num (manual)"
  else
    echo "PGs: autoscaler (on)"
  fi
  echo "CRUSH rule: $rule_name (root: $rule_root, failure domain: ${failure_type:-host})"
  echo "Replica size: $size"
  echo "Min replica size: $min_size"
  echo "Available ${failure_type:-host}s under root: ${available_domains}"
  echo

  read -rp "Create pool with these settings? (y/N): " confirm
  if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
    yellow "Pool creation cancelled."
    read -rp "Press Enter to continue..."
    return 0
  fi

  info "Creating pool '$pool_name'..."
  if [ -n "$pg_num" ]; then
    if ! ceph osd pool create "$pool_name" "$pg_num" replicated "$rule_name"; then
      red "Failed to create pool '$pool_name'."
      read -rp "Press Enter to continue..."
      return 1
    fi
  else
    if ! ceph osd pool create "$pool_name" 1 replicated "$rule_name" --autoscale-mode=on; then
      red "Failed to create pool '$pool_name'."
      read -rp "Press Enter to continue..."
      return 1
    fi
  fi

  green "Pool '$pool_name' created successfully!"
  ceph osd pool set "$pool_name" size "$size"
  ceph osd pool set "$pool_name" min_size "$min_size"

  # Enable application (important for metrics/health)
  read -rp "Enable application label (e.g., 'cephfs', 'rgw', 'rbd')? Leave blank to skip: " app_label
  if [ -n "$app_label" ]; then
    ceph osd pool application enable "$pool_name" "$app_label" --yes-i-really-mean-it || true
  fi

  echo
  info "Pool details:"
  ceph osd pool get "$pool_name" all || true

  # --- Test and optional clean-up preserving user's autoscaler preference ---
  echo
  info "Testing pool with a test object..."
  if echo "test-content" | rados -p "$pool_name" put test-object - 2>/dev/null; then
    green "Successfully wrote test object to pool!"
    ceph osd map "$pool_name" test-object || true

    echo
    yellow "Note: CephFS requires empty pools for metadata/data."
    echo "Choose how to clean the pool:"
    echo "  r) Remove only the test object (keeps all pool settings intact)"
    echo "  w) Wipe & recreate the pool (preserve PG/autoscaler mode, rule, size/min_size)"
    echo "  N) Do nothing"
    read -rp "Your choice [r/w/N]: " clean_choice

    if [[ "$clean_choice" == "r" || "$clean_choice" == "R" ]]; then
      info "Removing test object..."
      rados -p "$pool_name" rm test-object || true
      green "Test object removed. Pool is now empty."
    elif [[ "$clean_choice" == "w" || "$clean_choice" == "W" ]]; then
      info "Capturing current pool settings before wipe..."
      # Try JSON first, fall back to text parsing if jq or JSON not available
      if command -v jq >/dev/null 2>&1; then
        autoscale_mode=$(ceph osd pool get "$pool_name" pg_autoscale_mode -f json 2>/dev/null | jq -r '.pg_autoscale_mode // empty')
        pg_current=$(ceph osd pool get "$pool_name" pg_num -f json 2>/dev/null | jq -r '.pg_num // empty')
      fi
      if [ -z "${autoscale_mode:-}" ]; then
        autoscale_mode=$(ceph osd pool get "$pool_name" pg_autoscale_mode 2>/dev/null | awk -F': *' '/pg_autoscale_mode:/ {print $2}')
      fi
      if [ -z "${pg_current:-}" ]; then
        pg_current=$(ceph osd pool get "$pool_name" pg_num 2>/dev/null | awk '/pg_num:/ {print $2}')
      fi

      autoscale_mode=${autoscale_mode:-off}
      # Sensible default if not detected
      pg_current=${pg_current:-8}

      info "Wiping pool '$pool_name'..."
      ceph osd pool rm "$pool_name" "$pool_name" --yes-i-really-really-mean-it

      info "Recreating pool '$pool_name' (preserving settings)..."
      if [ "$autoscale_mode" = "on" ]; then
        ceph osd pool create "$pool_name" 1 replicated "$rule_name" --autoscale-mode=on
      else
        ceph osd pool create "$pool_name" "$pg_current" replicated "$rule_name"
      fi

      ceph osd pool set "$pool_name" size "$size"
      ceph osd pool set "$pool_name" min_size "$min_size"
      if [ -n "$app_label" ]; then
        ceph osd pool application enable "$pool_name" "$app_label" --yes-i-really-mean-it || true
      fi
      green "Pool '$pool_name' wiped and recreated empty (pg_autoscale_mode=$autoscale_mode, pg_num=$pg_current)."
    else
      yellow "Pool left as-is (contains the test object)."
    fi
  else
    yellow "Pool created but test write failed (PGs may still be activating)."
  fi
  # --- end test/cleanup ---

  read -rp "Press Enter to return to main menu..."
}


create_cephfs() {
  if ! ceph_ready; then
    red "Ceph cluster is not ready. Please initialize first."
    read -rp "Press Enter to continue..."
    return 1
  fi

  echo
  bold "Create Ceph File System (CephFS)"
  echo "================================"

  # List existing filesystems (if any)
  info "Existing CephFS (if any):"
  ceph fs ls || true
  echo

  # Inputs
  read -rp "Enter filesystem name (e.g., cephfs): " fs_name
  read -rp "Enter DATA pool name (e.g., fs_data): " data_pool
  read -rp "Enter METADATA pool name (e.g., fs_meta): " meta_pool
  read -rp "Number of MDS daemons (default: 1): " mds_count
  mds_count=${mds_count:-1}

  if [ -z "$fs_name" ] || [ -z "$data_pool" ] || [ -z "$meta_pool" ]; then
    red "Filesystem name, data pool, and metadata pool are required."
    read -rp "Press Enter to continue..."
    return 1
  fi

  # Check FS name not already in use
  if ceph fs ls -f json 2>/dev/null | grep -q "\"name\": *\"$fs_name\""; then
    red "A CephFS named '$fs_name' already exists."
    read -rp "Press Enter to continue..."
    return 1
  fi

  # Helper: ensure a pool exists; if not, create it with autoscaler
  ensure_pool() {
    local pool="$1"
    if ceph osd pool ls | awk '{print $1}' | grep -qx "$pool"; then
      info "Pool '$pool' already exists."
      return 0
    fi
    yellow "Pool '$pool' does not exist. Create it with autoscaler?"
    read -rp "Create pool '$pool'? (y/N): " ans
    if [[ "$ans" != "y" && "$ans" != "Y" ]]; then
      red "Cannot continue without pool '$pool'."
      return 1
    fi
    # Create minimal PGs and enable autoscaler; replicated with default CRUSH rule
    info "Creating pool '$pool' with autoscaler..."
    if ceph osd pool create "$pool" 1 replicated --autoscale-mode=on; then
      green "Pool '$pool' created."
    else
      red "Failed to create pool '$pool'."
      return 1
    fi
  }

  # Make sure both pools exist (create if missing)
  ensure_pool "$data_pool" || { read -rp "Press Enter to continue..."; return 1; }
  ensure_pool "$meta_pool" || { read -rp "Press Enter to continue..."; return 1; }

  # Enable cephfs application on pools (safe if repeated)
  info "Enabling 'cephfs' application on pools..."
  ceph osd pool application enable "$data_pool" cephfs --yes-i-really-mean-it || true
  ceph osd pool application enable "$meta_pool" cephfs --yes-i-really-mean-it || true

  echo
  bold "Configuration Summary:"
  echo "Filesystem name: $fs_name"
  echo "Data pool:       $data_pool"
  echo "Metadata pool:   $meta_pool"
  echo "MDS count:       $mds_count"
  echo
  read -rp "Proceed to create CephFS? (y/N): " confirm
  if [[ "$confirm" != "y" && "$confirm" != "Y" ]]; then
    yellow "Operation cancelled."
    read -rp "Press Enter to continue..."
    return 0
  fi

  # Create the filesystem
  info "Creating filesystem '$fs_name'..."
  if ceph fs new "$fs_name" "$meta_pool" "$data_pool"; then
    green "Filesystem '$fs_name' created."
  else
    red "Failed to create filesystem '$fs_name'."
    read -rp "Press Enter to continue..."
    return 1
  fi

  # Deploy MDS daemons via cephadm orchestrator
  info "Deploying $mds_count MDS daemon(s) for '$fs_name'..."
  if ceph orch apply mds "$fs_name" "$mds_count"; then
    green "Requested deployment of MDS for '$fs_name'."
  else
    yellow "Could not apply MDS via orchestrator. If you are not using cephadm, deploy MDS manually."
  fi

  echo
  info "Filesystem status:"
  ceph fs status "$fs_name" || true

  echo
  green "CephFS '$fs_name' is set up. You can now create a subvolume group / subvolumes or mount via ceph-fuse or kernel client."
  read -rp "Press Enter to return to the main menu..."
}


# ----------------- Menu -----------------
while true; do
  echo
  bold "Ceph Menu"
  echo "  1) Initialize cluster (steps 0–7, no OSDs)"
  echo "  2) Show device inventory"
  echo "  3) Add OSDs"
  echo "  4) Ceph Information"
  echo "  5) Check Chrony health (skew/offset)"
  echo "  6) make master as ntp server"
  echo "  7) crush map tools"
  echo "  8) create pool with specific rule"
  echo "  9) create new ceph file system"
  echo "  10) Exit"
  read -rp "Choose an option: " choice
  case "$choice" in
    1) initialize_cluster ;;
    2) show_inventory ;;
    3) add_osds ;;
    4) ceph_info_menu ;;
    5) check_chrony_health ;;
    6) ntp_set_master_and_clients ;;
    7) crush_map ;;
    8) create_pool_with_rule ;;
    9) create_cephfs ;;
    10) exit 0 ;;
    *) red "Invalid choice";;
  esac
done


```
