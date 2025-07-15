# Rethinking Container Persistence: Why I Ditched EBS Volumes for ECR

## Sometimes the simplest solution is already in your AWS account — you just need to look at it differently



Everyone says containers should be stateless. Need to save something? Use an EBS volume. That's the rule.

And yeah, for production databases and critical services running on AWS, that's right. But after dealing with internal tools and dev environments on ECS and EKS for years, I realized we were sometimes making things harder than they needed to be.

This is about a simpler approach I've used - snapshotting containers with docker commit and pushing them to Amazon ECR. Not for everything. Just for specific cases where EBS volumes felt like overkill.

## When EBS Volumes Are Too Much

You know the drill. Someone needs persistence, so you set up:

- EBS volumes or EFS file systems
- IAM policies for volume access
- Backup policies with AWS Backup
- Retention rules and lifecycle policies
- Lambda functions for cleanup

None of this is wrong. But for certain workloads - especially internal tools - the overhead can feel disproportionate.

Examples that come to mind:

- Browser-based IDEs on ECS
- Developer sandboxes on Fargate
- Training environments
- Preview or demo setups

The actual requirement is usually simple: "Let me come back to where I left off."

Not high availability. Not multi-AZ replication. Not long-term durability. Yet we still end up managing EBS volumes across task definitions.

## Storage Is Cheap, Complexity Isn't

Storage costs have dropped a ton. S3, ECR, EBS - they're all cheap now. Storing a few extra gigs isn't what wakes you up at 3am.

We know storage is cheap these days. But here's the real question: do we really need hard storage for everything?

You know what does keep you up?

- EBS volume attachment failures
- Orphaned EBS volumes accumulating costs
- ECS tasks stuck because volumes are in the wrong AZ
- Migrating volumes between ECS clusters or regions

So I started asking: if storage is cheap, why are we building all this infrastructure just to save some container state?

## Amazon ECR Is Already a Storage System

ECR already does a lot:

- S3-backed durable storage
- Layer deduplication
- Image compression
- IAM-based access control
- Lifecycle policies
- Cross-region replication

It's one of the most stable AWS services we use. So why not use it to store container state too?

## How docker commit Works Here

`docker commit` gets a bad rap, mostly because people misuse it. But it's actually perfect for this - it just captures a container's filesystem at a point in time.

    docker commit <container_id> <image:tag>
    docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>

Push it to ECR, and now that container state is:

- Immutable
- Versioned
- Portable

Exactly what you want.

## How It Actually Works

The flow is dead simple:

1. User starts a container, installs stuff, writes code
2. When they're done, commit the container and tag it with user/session info, then push to ECR
3. Next time they log in, pull the image from ECR and start from where they left off

No EBS volumes. No EFS mounts. No cross-AZ headaches. Just ECR images.

## Real Example: Browser IDEs on ECS

I used this for a browser-based IDE platform running on ECS Fargate. Each developer got their own container. When they logged out, we'd commit and push to ECR. When they logged back in, we'd pull from ECR and start from that exact state.

This fixed a bunch of problems:

- Zero EBS-related incidents
- Easy rollbacks (just use an older ECR image tag)
- Cleanup happened automatically via ECR lifecycle policies
- No cross-AZ volume attachment issues
- New developers onboarded way faster

Best part? Fewer AWS resources to manage means fewer things that can break.

## Why This Is Easier to Operate on AWS

The best part isn't cost or performance - it's predictability.

ECR already handles:

- Image scanning for vulnerabilities
- Tag-based lifecycle policies
- Time-based retention
- Cross-region replication

No orphaned EBS volumes. No surprise storage costs. And bonus: moving workloads between ECS clusters or even to EKS becomes trivial since everything's just an ECR image.

## Don't Use This For Everything

Seriously, don't use this for:

- RDS databases or DocumentDB
- High-write workloads
- Shared state between ECS services
- Anything with strict compliance requirements

ECR images can get large. You need lifecycle policies. Never bake AWS credentials or secrets into images - use AWS Secrets Manager or Parameter Store instead.

This only makes sense when:

- State is per-user
- It's somewhat ephemeral
- You could rebuild it if needed

## Final Thoughts

EBS volumes and EFS are still right for most things. But not every persistence problem needs the same solution.

For internal tools and dev environments on ECS or EKS, snapshotting to ECR is often simpler and more reliable than managing volumes across availability zones.

Sometimes the best AWS infrastructure is the stuff you're already paying for.

---

Have you tried something similar in your environment? I'd love to hear about your experiences - whether this approach worked for you or if you found other creative solutions. Drop a comment below or reach out. The best infrastructure patterns often come from sharing what actually works in the real world.

And if you're thinking "but wait, what about..." - those are exactly the questions worth discussing. Let's talk about it.
