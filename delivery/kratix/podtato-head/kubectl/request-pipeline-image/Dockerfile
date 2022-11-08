FROM "mikefarah/yq:4"

RUN [ "mkdir", "/tmp/transfer" ]
ADD manifest.yaml /tmp/transfer/manifest.yaml
ADD change-to-cluster-ip change-to-cluster-ip
ADD prepend-resource-names prepend-resource-names

CMD [ "sh", "-c", "./change-to-cluster-ip; ./prepend-resource-names" ]
ENTRYPOINT []