Describe the cluster and use `grep` to help you find the Easter egg node label:

```bash
gcloud container clusters describe newclusterwhodis --region us-central1-b | grep labels -i -A1 -B5
```

Heck yes you did it! You're friggin awesome!