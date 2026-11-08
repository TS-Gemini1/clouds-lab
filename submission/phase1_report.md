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
50.06
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
| IaaS VM | <!--answer:vm_cold_ms-->253.5<!--/answer--> | <!--answer:vm_warm_median_ms-->250.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1508.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->261.9<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1562.2<!--/answer--> | <!--answer:function_warm_median_ms-->261.6<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  253.5ms  warm_median=  250.8ms
            ######################################
[OK  ] cloudrun  cold= 1508.1ms  warm_median=  261.9ms
            ########################################
[OK  ] function  cold= 1562.2ms  warm_median=  261.6ms
            ########################################
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
The VM creates nothing automatically: one e2-micro (2 shared vCPUs, 1 GB RAM) runs gunicorn with 1 worker and 8 threads, so ten or a hundred requests just queue and slow down. Its hard limit is the machine size I picked. Cloud Run starts new container instances when requests pile up and scales back to zero when on idle; its limit is the --max-instances=3 flag I passed. The Function behaves the same way, because it runs as a Cloud Run service, but with a concurrency of 1 request per instance, so it needs more instances for the same load. Its maximum is Google’s default, since I set none.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM took the most work. I built a custom VPC, a subnet and two firewall rules, created the VM with a network tag, gave its service account read access to Artifact Registry, SSHed in, installed Docker and started the container by hand. I would also have to maintain the OS and restarts, and it costs money while it runs. Cloud Run needed a Dockerfile and an image built with Cloud Build, then a single deploy command; Google handles servers, HTTPS, scaling and restarts. The Function was the least work: no Dockerfile or image, just the source folder and an entry point. Google built the container itself.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
   SERVICE   REGION   URL                                       
   LAST DEPLOYED BY                                            
   LAST DEPLOYED AT
+  echo      us-central1  https://echo-989841407709.us-central1.run.app      service-989841407709@gcf-admin-robot.iam.gserviceaccount.com  2026-11-08T21:19:40.238135Z
+  echo-bot  us-central1  https://echo-bot-989841407709.us-central1.run.app  theodoros.syriopoulos@eurecom.fr                              2026-11-08T21:14:34.053704Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output lists two Cloud Run services: echo-bot, which I deployed, and echo, deployed by the gcf-admin-robot account. A gen-2 Cloud Function is not a separate runtime: Google builds my source into a container image (stored in gcf-artifacts) and runs it as a Cloud Run service. So on a cold start both do the same work: schedule a container, start Python and load the app. That’s why my cold starts are so close (1508 ms vs 1562 ms) and the warm medians are almost identical (261.9 vs 261.6 ms). What genuinely differs is the deployment path and defaults: Cloud Run received my own image built from my Dockerfile, while the Function received source code and was built with Google’s buildpacks. The Function also defaults to concurrency 1 and a smaller instance (0.17 vCPU, 256 MB). The VM has no cold start at all, because it is always running.
<!--/answer-->
