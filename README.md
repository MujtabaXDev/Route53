# Route 53 Lab Site

A tiny static site that shows **which region served your request**, plus a live
failover watcher. Deploy the same files in two regions, then let Route 53 steer
traffic between them.

## Files

| File | Purpose |
|---|---|
| `index.html` | The page: region/AZ/instance card, failover watcher, dig commands |
| `health.txt` | Contains `OK`. Target for Route 53 health checks |
| `user-data.sh` | EC2 bootstrap: installs nginx, writes `/whoami.json`, pulls the site |

`/whoami.json` is generated per instance (not in the repo), so each region reports itself.

## Preview locally

Open `index.html` in a browser. It detects `file://` and shows simulated data.

## Deploy (per region)

1. Upload the site files to S3 (or push them to a Git repo):
   `aws s3 sync . s3://YOUR-BUCKET/route53-lab/ --exclude "user-data.sh" --exclude "README.md"`
2. Edit `SITE_SOURCE` in `user-data.sh`.
3. Launch an EC2 instance (Amazon Linux 2023, t3.micro) with this user-data. Give it an
   instance profile that can read the bucket if you use S3. Security group: allow HTTP 80
   (from the ALB, or from anywhere for a first test).
4. Put the instance behind an ALB (recommended), or use its public IP directly with an
   A record while you learn.
5. Repeat in a second region (for example `us-east-1` and `eu-west-1`).

## Route 53 setup

- **Health check:** HTTP, path `/health.txt`, enable string matching on `OK`.
- **Records:** create the same name (e.g. `app.yourdomain.com`) twice, once per region,
  with the routing policy you want to test: weighted, latency, failover, or geolocation.
  Attach each record's health check. Use alias records when pointing at an ALB.
- Start with a low TTL (e.g. 30-60 s) so changes are easy to observe.

## Testing failover

On the server in the primary region:

```bash
sudo mv /usr/share/nginx/html/health.txt /usr/share/nginx/html/health.off
```

The page still works, but the health check now gets a 404 and fails. After the check marks
the endpoint unhealthy (interval x failure threshold) and the TTL expires, new lookups go to
the other region and the watcher logs `Switched: region-a → region-b`.
Restore with the reverse `mv`.

Other experiments: `sudo systemctl stop nginx` (hard outage), change weights 90/10 and count
results with `for i in $(seq 20); do curl -s app.yourdomain.com/whoami.json; echo; done`
(note: reuse of cached DNS answers can hide the split, so try `dig` first or use different resolvers).

## Clean up

Delete health checks, hosted zone (if no longer needed), ALBs, and EC2 instances in both
regions when done to avoid charges.
