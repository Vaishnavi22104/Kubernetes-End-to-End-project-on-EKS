# Play 2048 on AWS Kubernetes (EKS)

In this project we put the **2048 game** on the internet using **Amazon EKS** (Kubernetes on AWS).

When you finish, you get a web link. Open it in your browser and play the game.


---

## The big idea (in simple words)

Think of a restaurant:

| In the restaurant | In this project | What it does |
|---|---|---|
| The kitchen | **EKS cluster** | The place where everything runs |
| The cooks | **Pods** (on Fargate) | They run the 2048 game |
| The waiter | **Service** | Knows which cooks are available |
| The front door | **Ingress + Load Balancer (ALB)** | Lets people from the internet come in |
| The door manager | **AWS Load Balancer Controller** | Builds the front door for us |

The journey of one visitor:

```
You (browser) → Load Balancer → Service → Pods running the game
```

---

## What you need first

Install these 4 tools on your computer. Guide: [prerequisites.md](prerequisites.md)

| Tool | Why we need it |
|---|---|
| **AWS CLI** | Talks to your AWS account |
| **kubectl** | Talks to Kubernetes |
| **eksctl** | Builds the EKS cluster for us |
| **Helm** | Installs apps into Kubernetes |

You also need an AWS account. Run `aws configure` once to log in.

**Warning: this costs money** (about **$0.30 per hour** while it runs). Delete everything when you finish. See the last step.

---

## Step by step

We use the cluster name `demo-cluster` and the region `us-east-1`.

### Step 1: Create the cluster
This builds your "kitchen". It takes about 15–20 minutes.
```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
More details: [installing-eks.md](installing-eks.md)

### Step 2: Allow the game to run
Fargate only runs pods in places you allow. This allows the `game-2048` area.
```bash
eksctl create fargateprofile --cluster demo-cluster --region us-east-1 \
  --name alb-sample-app --namespace game-2048
```

### Step 3: Deploy the game
This creates the game pods, the Service and the Ingress.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
Check the pods (wait until they say `Running`):
```bash
kubectl get pods -n game-2048
```
More details: [2048-app-deploy-ingress.md](2048-app-deploy-ingress.md)

### Step 4: Give the cluster an ID card (OIDC)
This lets Kubernetes safely ask AWS for permissions.
```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --region us-east-1 --approve
```
More details: [configure-oidc-connector.md](configure-oidc-connector.md)

### Step 5: Install the "door manager" (Load Balancer Controller)
Without it, the Ingress does nothing. Follow every command in
[alb-controller-add-on.md](alb-controller-add-on.md). In short:
1. Download the permission file and create an IAM policy.
2. Create an IAM role and service account with `eksctl`.
3. Install the controller with `helm`. Fargate needs `region` and `vpcId`, so add both.

Check that it is running (you should see `2/2`):
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Step 6: Open the game
```bash
kubectl get ingress -n game-2048
```
Copy the long link under **ADDRESS**, paste it in your browser and play.
(Wait 2–3 minutes if it doesn't open at first.)

---

## Clean up

Do it **in this order**, or AWS keeps charging you.

```bash
# 1. Delete the game first (this also removes the Load Balancer)
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

# 2. Remove the controller
helm uninstall aws-load-balancer-controller -n kube-system

# 3. Delete the whole cluster
eksctl delete cluster --name demo-cluster --region us-east-1
```

---

## Something not working?

| Problem | What to do |
|---|---|
| Pods stay `Pending` | Run Step 2 (the Fargate profile) |
| ADDRESS is empty | Check the controller from Step 5 is running |
| Controller keeps crashing | You forgot `region` and `vpcId` in the Helm command |
| Link doesn't open | Wait a few minutes; the Load Balancer needs time to start |

---

## What's in this repo

| File | What it explains |
|---|---|
| [prerequisites.md](prerequisites.md) | Tools to install |
| [installing-eks.md](installing-eks.md) | Create and delete the cluster |
| [2048-app-deploy-ingress.md](2048-app-deploy-ingress.md) | Deploy the game |
| [configure-oidc-connector.md](configure-oidc-connector.md) | Set up the cluster ID card |
| [alb-controller-add-on.md](alb-controller-add-on.md) | Install the Load Balancer Controller |
| [sample-app.md](sample-app.md) | Extra practice with a simple nginx app |
