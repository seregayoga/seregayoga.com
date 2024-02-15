+++
title = 'How to host Hugo website on Github Pages using custom domain name'
date = 2024-02-15T23:38:04+01:00
draft = false
+++

This website https://seregayoga.com is built on Hugo static site generator and hosted on Github. There are two tutorials from [Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/) and from [Github](https://gohugo.io/hosting-and-deployment/hosting-on-github/) on how to configure it. Here is the gist of them on how to do the same on your own domain name. Assuming it is `example.com` and your Github profile name is `your-github-profile`.

1. Create a Hugo website. You can take a look at official [quickstart guide](https://gohugo.io/getting-started/quick-start/).
2. Create a public Github repo and push the code there from the previous step.
3. [Verify your custom domain.](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages) At this step you will be adding a `TXT` DNS record. Do not close web interface of your DNS provider; you will need it for the next step.
4. Continuing configuring DNS: add four `A` records to point your domain to Github servers:
```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```
Additionally, if your DNS provider supports IPv6, add four AAAA records:
```
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153
```
And finally add a `CNAME` record with name `www.example.com` and value `your-github-profile.github.io.`; by doing this you will have a redirect from `www.example.com` to `example.com`.

Verify you did everything right by following `dig` commands:
```
$ dig www.example.com +noall +answer -t A
www.example.com.	2872	IN	CNAME	your-github-profile.github.io.
your-github-profile.github.io.	2872	IN	A	185.199.111.153
your-github-profile.github.io.	2872	IN	A	185.199.109.153
your-github-profile.github.io.	2872	IN	A	185.199.108.153
your-github-profile.github.io.	2872	IN	A	185.199.110.153

$ dig www.example.com +noall +answer -t AAAA
www.example.com.	2869	IN	CNAME	your-github-profile.github.io.
your-github-profile.github.io.	2869	IN	AAAA	2606:50c0:8000::153
your-github-profile.github.io.	2869	IN	AAAA	2606:50c0:8001::153
your-github-profile.github.io.	2869	IN	AAAA	2606:50c0:8003::153
your-github-profile.github.io.	2869	IN	AAAA	2606:50c0:8002::153

$ dig www.example.com +noall +answer -t CNAME
www.example.com.	2866	IN	CNAME	your-github-profile.github.io.

```
5. Go to settings of your Github repository, then `Pages` and in `Build and deployment` section in the `Source` dropdown choose `Hugo` and click on `Configure`. Github will provide you with a Github Actions config file, simply commit these changes to your repository.
6. At this step you might already close `Pages` settings page, if so open it again. There in `Custom domain` specify yours, in our case it will be `example.com`. You should see `DNS check successful` badge meaning you did everything right in steps 3. and 4. Make sure `Enforce HTTPS` checkbox is checked so Github will take care of enabling https.
7. Voilà! Now you can enjoy your website powered by Github Pages on your own domain.