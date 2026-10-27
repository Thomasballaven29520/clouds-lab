# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->256.0<!--/answer--> | <!--answer:vm_warm_median_ms-->212.0<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->3845.5<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->661.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->2289.7<!--/answer--> | <!--answer:function_warm_median_ms-->644.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
Phase 1 — deployment measurements (measure.py, 20 warm requests each)
[OK  ] vm        cold=  256.0ms  warm_median=  212.0ms
            #############
[OK  ] cloudrun  cold= 3845.5ms  warm_median=  661.9ms
            ########################################
[OK  ] function  cold= 2289.7ms  warm_median=  644.8ms
            #######################################
````
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
VM: nothing is created automatically. One e2-micro (2 shared vCPUs, 1 GB RAM) serves every request; with 100 concurrent requests they queue behind gunicorn's 8 threads and latency grows. The limit is the machine size I chose. Cloud Run: new container instances of my image are started automatically as concurrency rises (up to 80 requests per instance by default) and removed down to zero when idle; the hard cap is my --max-instances=3. Function: same mechanism, but each instance takes only 1 request at a time (maxInstanceRequestConcurrency=1), so it scales out sooner and reaches its --max-instances=3 cap sooner.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM (IaaS): the most work. I built a custom VPC, a subnet and two tagged firewall rules (8080 and 22), created the instance, granted its service account Artifact Registry read access, SSHed in, installed Docker, configured registry auth and ran the container. I also own OS patching, restarts and the external IP, which is billed while it exists. Cloud Run (PaaS): I wrote a multi-stage Dockerfile and built the image once with Cloud Build; deploying was a single command with no network or OS to manage. Function (FaaS): the least effort, no Dockerfile at all: I deployed the source folder with one command and Google built the container for me.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-279859701913.us-central1.run.app
LAST DEPLOYED BY: service-279859701913@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-18T11:26:01.014428Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-279859701913.us-central1.run.app
LAST DEPLOYED BY: Thomas.ballaven@eurecom.fr
LAST DEPLOYED AT: 2026-10-18T11:22:39.888902Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The list shows two Cloud Run services although I deployed only one: "echo", deployed by gcf-admin-robot, is my Cloud Function. A gen-2 Function is a Cloud Run service under the hood: Cloud Build turns my source into a container image (stored in the gcf-artifacts repository) and it runs on the same Cloud Run infrastructure with the same scale-to-zero model. That is why both cold starts are in the seconds (3.8 s and 2.3 s) while the always-on VM shows ~0.26 s: both must start a container instance from zero, then Python and the web server. What genuinely differs is the deployment path and defaults: the Function is built from source by a Google buildpack with Functions Framework instead of my Dockerfile, gets smaller resources (0.17 vCPU, 256 MB) and serves one request per instance instead of 80.
<!--/answer-->
