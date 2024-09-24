do-release-upgrade
y
y
RETURN
RETURN
N # timesync config file
install the package maintainer file # openssh
install the package maintainer file # grub

y # remove obsolete packages
y # reboot

# Post reboot fixes:

echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc > /etc/apt/trusted.gpg.d/postgres.asc
apt update

apt --fix-broken install -y
apt upgrade -y

apt install libvips -y

runuser -l postgres -c "psql -c 'ALTER DATABASE runner REFRESH COLLATION VERSION;'"
runuser -l postgres -c "psql -c 'ALTER DATABASE postgres REFRESH COLLATION VERSION;'"
runuser -l postgres -c "psql -c 'ALTER DATABASE template1 REFRESH COLLATION VERSION;'"
