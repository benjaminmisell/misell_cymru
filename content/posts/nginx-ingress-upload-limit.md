---
date: "2018-01-31T09:03:04Z"
title: "Increasing the upload limit on Kubernetes NGINX Ingress"
authors: []
tags:
  - kubernetes
  - nginx
draft: false
---

This is another quick post (I quite like this type of post) detailing how I increased the upload file size limit on the kubernetes nginx ingress controller. I decided to write this because I found a lack of decent documentation on the subject and I thought if at least one person finds this useful I have done good. 

So my first attempt using some 5 minute Googling did not work. I said i could set `    "body-size": "0"` on the nginx config map to have no limit, but that seemed to have no effect. Ah well, back to the infinite source of wisdom and lies that is the Internet.

I also didn't really want this set globally, I'd prefer per ingress definition. Another *half an hour* later I found [this](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/customization) page showing how it's actually meant to be done. The first thing I found out was that I should have been setting `client-max-body-size` not `body-size`, thank you random incorrect blog post! The second thing was that I could set the `nginx.org/client-max-body-size` annotation on the ingress definition (yay!).

So with my newly found knowledge I can now add this to my ingress, in the metadata section:

```yaml
annotations:
	nginx.org/client-max-body-size: "0"
```
And done! Tested (always test) and working. I hope you found this useful in some way.