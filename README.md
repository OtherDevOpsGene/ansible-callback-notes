# ansible-callback-notes

A test environment for testing/demonstrating Ansible Provisioning Callbacks for
AWS autoscaling.

I stood up a new AWS EKS cluster using `eksctl`, used the `awx-operator` to
install AWX (open-source Ansible Tower) into the cluster, and then used
Terraform to stand up an EC2 instance in an auto-scaling group (ASG) with
a provisioning callback in the userdata. I put load on that instance to trigger
the auto-scaling and made sure the newly scaled instance "phoned home" to AWX.

## eksctl

* [eksctl](https://eksctl.io/)

Per the `eksctl` [minimum IAM policies](https://eksctl.io/usage/minimum-iam-policies/),
the AWS user needs:

* AmazonEC2FullAccess
* AWSCloudFormationFullAccess
* `eksctl-iam-policy.json` (replace `054858005475` with the account number)

Just having those caused an error:

```
Error: unable to determine AMI to use: error getting AMI from SSM Parameter Store: AccessDeniedException: User: arn:aws:iam::054858005475:user/gene.gotimer-eks is not authorized to perform: ssm:GetParameter on resource: arn:aws:ssm:us-east-2::parameter/aws/service/eks/optimized-ami/1.20/amazon-linux-2/recommended/image_id
        status code: 400, request id: 2a4a86a1-0053-4ef1-bcb7-16b8b256e8d0. please verify that AMI Family is supported
```

so I added `AmazonSSMFullAccess`, even though the [IAM Policy Simulator](https://policysim.aws.amazon.com/home/index.jsp?#)
said I shouldn't need it.

```bash
eksctl create cluster -f cluster.yaml
eksctl create nodegroup --cluster=gene-sandbox --nodes-min=2 --nodes-max=5 --instance-types=m5.xlarge ng-1
```

## AWX operator

* [ansible/awx-operator](https://github.com/ansible/awx-operator)

```bash
git clone git@github.com:ansible/awx-operator.git
cd awx-operator
kubectl apply -f awx-demo.yml
kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"
kubectl port-forward services/awx-demo-service :80 &
```

The AWX operator needed the `m5.xlarge` instances in the nodegroup. With just
`m5.large`, the `aws-demo` pods were stuck in `Pending` with a warning
`Insufficient cpu.`

Use `service_type: LoadBalancer` so Ansible will have a URL to hit for the AWX
server. It will take a few minutes to be up and running. Then you can log in
with username `admin`. Get the admin password from:

```bash
kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" | base64 --decode
```

## Terraform ASG

## Ansible provisioning callbacks

* [Provisioning callbacks](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html#provisioning-callbacks)
