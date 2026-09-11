# What can the research cluster actually run for this tool?

Type: task
Status: claimed
Blocked by: 

## Question

HITL task for the owner. Needed facts before hosting can be decided: GPU type and VRAM available; scheduler (Slurm, Kubernetes, plain SSH); whether a long-lived service or only batch jobs can run; outbound internet access from jobs (needed for Anthropic API and email); where files can be stored persistently and who can read them; whether a small web page can be exposed to the lab network. Record the answers in the ticket. Checklist for the owner: run nvidia-smi on a node, note the scheduler, try curl https://api.anthropic.com from a job, note the storage path.
