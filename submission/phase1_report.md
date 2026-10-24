# Phase 1 — Comparison Report

Echo bot deployed three ways (IaaS VM, Cloud Run, Cloud Function gen 2) in `us-central1`,
measured from Google Cloud Shell with `measure.py` (1 cold request + 20 sequential warm
requests per target). All latency figures below are copied from `submission/phase1_report.json`.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
47.7
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

`echo-net` is a custom-mode VPC with one subnet, `echo-subnet` (`10.10.0.0/24`, `us-central1`),
and two ingress rules, `allow-echo-8080` (tcp:8080) and `allow-echo-ssh` (tcp:22), both with
`--target-tags=echo-http`. The tag is what makes a rule apply to an instance: a custom VPC starts
with only the implied allow-egress / deny-ingress rules, so a VM without the `echo-http` tag
would receive no traffic on either port, even though the rules exist.

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->253.9<!--/answer--> | <!--answer:vm_warm_median_ms-->211.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->3095.2<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->130.2<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->3009.3<!--/answer--> | <!--answer:function_warm_median_ms-->130.6<!--/answer--> |

<!--answer:latency_plot-->
```
measure.py summary
------------------------------------------------------------
[OK  ] vm        cold=  253.9ms  warm_median=  211.8ms
            ########################################
[OK  ] cloudrun  cold= 3095.2ms  warm_median=  130.2ms
            #########################
[OK  ] function  cold= 3009.3ms  warm_median=  130.6ms
            #########################
------------------------------------------------------------

Histogram of the 20 warm requests per platform (count per bin)
bin (ms)     vm                     cloudrun               function
100-150                         0   #################  17  ################## 18
150-200                         0                       0                      0
200-250      #################### 20                    0                      0
250-600                         0                       0                      0
600-700                         0   ###                 3  ##                  2

warm stats (ms)   min     median   p95     max
vm                211.0   211.8    213.3   214.3
cloudrun          121.4   130.2    649.4   681.9
function          122.8   130.6    637.7   642.5
```

The VM has no meaningful cold start: its container never stops, so its "cold" request (253.9 ms)
is close to its warm median (211.8 ms). Cloud Run and the Function were measured after a full
night idle, so both had scaled to zero and both cold numbers (~3 s) include a real instance start.
Warm requests on the VM are very stable (3 ms spread). Cloud Run and the Function share the same
bimodal shape: most requests at ~120–140 ms, a few at ~640–680 ms, which is why their p95 is ~5×
their median.
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
- **VM:** Creates nothing automatically. One, ten, or a hundred requests hit the same single container instance. Excess load simply queues at the OS/socket level. The hard limit is physical, determined by the chosen machine size (`--machine-type=e2-micro`).
- **Cloud Run:** Automatically scales out container instances as concurrent requests rise. The hard limit comes from the flag we explicitly passed (`--max-instances=3`), beyond which further requests queue or get dropped.
- **Cloud Function (gen 2):** Also scales container instances out automatically under load. Its limit is governed by the function's max-instance configuration, which by default is 100 instances.
<!--/answer-->

<!--answer:deployment_effort-->
**VM (most effort):** 
I built the network myself (custom VPC, subnet, two firewall rules with a
target tag), created the instance, granted its service account `artifactregistry.reader`, opened a terminal inside the VM through SSH,
installed Docker, configured registry authentication and ran the container with a restart policy. If this were a real service, I would also have to install the latest Debian security updates, update Docker, and restart the VM or the container if they stop working. Moreover, the VM runs 24/7 whether or not it gets traffic, so its external IP is billed the whole time.

**Cloud Run:** I wrote the multi-stage Dockerfile, built and pushed the image once. The command
`gcloud run deploy` gave me: HTTPS, a public URL, autoscaling and scale-to-zero. Nothing to
maintain but the image.

**Function (least effort):** no Dockerfile at all: I ran the command `gcloud functions deploy` with the
source folder and an entry point; Google built and hosted the container for me.
There was a setup snag shared by both build paths (the Dockerfile image build used by the VM and
Cloud Run, and the Function's source build): Cloud Build runs as the Compute Engine default
service account, which needed `roles/cloudbuild.builds.builder` before it could read the uploaded
source (my first image build failed with a 403 until I granted it).
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-550165757637.us-central1.run.app
LAST DEPLOYED BY: service-550165757637@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-22T23:35:16.251575Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-550165757637.us-central1.run.app
LAST DEPLOYED BY: Giorgio.Montini@eurecom.fr
LAST DEPLOYED AT: 2026-10-22T23:21:35.468215Z
```
<!--/answer-->

<!--answer:convergence_explanation-->
Running `gcloud run services list` reveals that the Cloud Function is listed as an active Cloud Run service, `echo`, deployed not by me but by the Cloud Functions service agent (`gcf-admin-robot`). This demonstrates that Cloud Functions (gen 2) is not an independent execution environment, but an abstraction layer running directly on top of Cloud Run.

Because both endpoints share the same serving infrastructure, scale-to-zero mechanism and Google Front End routing, their numbers are nearly identical (cold 3095 vs 3009 ms, warm median 130.2 vs 130.6 ms): both must start a container instance from zero and initialize the Python runtime before serving the first request.

What genuinely differs is the packaging and deployment path: Cloud Run requires me to write and build my own Docker image, whereas Cloud Functions accepts raw source code and builds the image itself with Cloud Build (buildpacks, stored in `gcf-artifacts`). The defaults also differ: the Function serves one request per instance with 0.17 vCPU and 256 MB, while the Cloud Run service gets 1 vCPU and up to 80 concurrent requests per instance.
<!--/answer-->
