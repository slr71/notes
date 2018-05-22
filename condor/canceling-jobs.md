``` shell
for uuid in aab43f36-6808-4e66-a6b0-4b9637d8af71; do
    sudo docker run --net=host \
        -v /etc/jobservices.yml:/etc/jobservices.yml \
        --rm discoenv/de-job-killer:master \
        -kill -uuid $uuid -config /etc/jobservices.yml
done
```
