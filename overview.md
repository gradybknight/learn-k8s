# Kubernetes Learning Platform

I would like to learn how to setup and administer a kubernetes cluster. To that end, I've purchase three mini pcs (details below). I would like to create a series of markdown instructions for setting up the machines for a k8s cluster.

I have propsed a syllabus for setting up the cluster and initial learning. Please review the proposed breakdown and make updates

## Proposed breakdown of instructions

1. Install a linux server operating system on each machine
   1. Determine appropriate linux variant
   1. Steps to create an ISO
   1. Install and configure each indivual box
   1. Configure knowing that each box will be connected to a switch. the switch will connect to a router
1. Install kubernetes software.
   1. Lenovo box is the most powerful
1. Build a minimal backend application
   1. Node/Typescript app serving `/health` routes and `/hello` route.
   1. Connect to a database housed in k8s cluster
1. Build a minimal frontend application
   1. React/Typescript app which hits backend `/hello` route
   1. Deploy as a Single Page App through k8s cluster
1. Determine how to expose k8s cluster to "outside world"
   1. Running on gooble fiber in house. No guarantee of static IP address
   1. Evaluate different solutions
1. Learn "git ops" with cluster
   1. How to trigger CI/CD with push to different git branches
   1. Determine if argo cd makes sense

## Machine Specs

### Lenovo box (one)

- Lenovo ThinkCentre M720Q Tiny Desktop Intel i5-8500T up to 3.50GHz 16GB RAM 256GB SSD, Wi-Fi, Bluetooth, Keyboard & Mouse

### Beelink boxes (two)

- Beelink Mini PC, Mini S12 Pro Intel 12th N100(Up to 3.4GHz), 16GB DDR4 500GB M.2 SSD, Desktop Computers Support 4K Dual HDMI Display/WiFi6/BT5.2/USB3.2/1000Mbps LAN/WOL Home/Office
