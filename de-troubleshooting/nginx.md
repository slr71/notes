### Troubleshooting nginx Connection Issues

- Check nginx.conf within the nginx container to verify that it's displaying the correct color.
- Verify that the IP address that it lists matches the actual container.
- Send a SIGHUP to nginx to consul-template and nginx to make sure that they have a chance to update their configs.
- Review `/var/log/de/nginx/ui-nginx.log`.
