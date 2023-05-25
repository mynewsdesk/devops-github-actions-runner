# Hetzner installimage

It's possible to create a reusable image to use with the `installimage` tool.

NOTE: We don't use this approach at the moment but may use it in the future. A limiting factor is the time it takes to transfer the ~2GB image over the network. Maybe using a minimal Ubuntu image would be better.

```bash
# First remove any excess kernels (Hetzner allows only one to be present in the final image)
apt search linux-image | grep installed
apt remove linux-image-... -y
apt --purge autoremove -y

# Remove any previous installimage configuration
rm /installimage*

apt install pigz -y
tar cf - --exclude=/dev --exclude=/proc --exclude=/sys --exclude=/ubuntu-2204-base-amd64.tar.xz / | pigz -9 > /ubuntu-2204-github-actions-runner-amd64.tar.xz
```

Then use `scp` to copy the image over.

References:
- https://keithtenzer.com/cloud/how-to-create-a-rhel-8-image-for-hetzner-root-servers/