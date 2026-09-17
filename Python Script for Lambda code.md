import boto3
import json
from datetime import date, datetime, timedelta, timezone

S3_BUCKET = "aws-cost-optimization-123"

SNS_TOPIC_ARN = "arn:aws:sns:us-east-1:716245436026:aws-cost-optimization-alerts"


def get_regions():
    ec2 = boto3.client("ec2", region_name="us-east-1")

    response = ec2.describe_regions(
        AllRegions=False
    )

    return [
        region["RegionName"]
        for region in response["Regions"]
    ]


def get_costs():
    ce = boto3.client(
        "ce",
        region_name="us-east-1"
    )

    today = date.today()
    current_month = today.replace(day=1)

    start_month = (
        current_month - timedelta(days=180)
    ).replace(day=1)

    end_date = today + timedelta(days=1)

    response = ce.get_cost_and_usage(
        TimePeriod={
            "Start": start_month.isoformat(),
            "End": end_date.isoformat()
        },
        Granularity="MONTHLY",
        Metrics=["UnblendedCost"],
        GroupBy=[
            {
                "Type": "DIMENSION",
                "Key": "SERVICE"
            }
        ]
    )

    monthly_costs = []

    for period in response["ResultsByTime"]:

        month = period["TimePeriod"]["Start"]
        services = []

        for group in period["Groups"]:

            service = group["Keys"][0]

            cost = float(
                group["Metrics"]["UnblendedCost"]["Amount"]
            )

            services.append({
                "service": service,
                "cost": round(cost, 4)
            })

        services.sort(
            key=lambda x: x["cost"],
            reverse=True
        )

        total = round(
            sum(item["cost"] for item in services),
            4
        )

        monthly_costs.append({
            "month": month,
            "total_cost": total,
            "services": services
        })

    return {
        "months": monthly_costs
    }


def get_ec2_cpu(instance_id, region):

    cloudwatch = boto3.client(
        "cloudwatch",
        region_name=region
    )

    end_time = datetime.now(timezone.utc)

    start_time = (
        end_time - timedelta(days=7)
    )

    response = cloudwatch.get_metric_statistics(
        Namespace="AWS/EC2",
        MetricName="CPUUtilization",
        Dimensions=[
            {
                "Name": "InstanceId",
                "Value": instance_id
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=86400,
        Statistics=["Average"]
    )

    datapoints = response.get(
        "Datapoints",
        []
    )

    if not datapoints:
        return None

    averages = [
        point["Average"]
        for point in datapoints
    ]

    return round(
        sum(averages) / len(averages),
        2
    )


def get_resources():

    resources = {
        "ec2": [],
        "ebs": [],
        "rds": [],
        "nat_gateways": [],
        "lambda": [],
        "ecs": [],
        "elb": [],
        "s3": []
    }

    regions = get_regions()

    for region in regions:

        print(
            f"Scanning region: {region}"
        )

        ec2 = boto3.client(
            "ec2",
            region_name=region
        )

        # EC2
        response = ec2.describe_instances(
            Filters=[
                {
                    "Name": "instance-state-name",
                    "Values": ["running"]
                }
            ]
        )

        for reservation in response["Reservations"]:

            for instance in reservation["Instances"]:

                instance_id = instance["InstanceId"]

                cpu_average = get_ec2_cpu(
                    instance_id,
                    region
                )

                resources["ec2"].append({
                    "region": region,
                    "instance_id": instance_id,
                    "instance_type": instance["InstanceType"],
                    "state": instance["State"]["Name"],
                    "cpu_average_7_days": cpu_average
                })

        # EBS
        volumes = ec2.describe_volumes()

        for volume in volumes["Volumes"]:

            resources["ebs"].append({
                "region": region,
                "volume_id": volume["VolumeId"],
                "size_gb": volume["Size"],
                "state": volume["State"],
                "attached": len(
                    volume.get("Attachments", [])
                ) > 0
            })

        # NAT Gateway
        nat = ec2.describe_nat_gateways(
            Filters=[
                {
                    "Name": "state",
                    "Values": ["available"]
                }
            ]
        )

        for gateway in nat["NatGateways"]:

            resources["nat_gateways"].append({
                "region": region,
                "nat_gateway_id": gateway["NatGatewayId"],
                "state": gateway["State"]
            })

        # RDS
        rds = boto3.client(
            "rds",
            region_name=region
        )

        dbs = rds.describe_db_instances()

        for db in dbs["DBInstances"]:

            resources["rds"].append({
                "region": region,
                "db_identifier": db[
                    "DBInstanceIdentifier"
                ],
                "instance_class": db[
                    "DBInstanceClass"
                ],
                "status": db[
                    "DBInstanceStatus"
                ]
            })

        # Lambda
        lam = boto3.client(
            "lambda",
            region_name=region
        )

        paginator = lam.get_paginator(
            "list_functions"
        )

        for page in paginator.paginate():

            for function in page["Functions"]:

                resources["lambda"].append({
                    "region": region,
                    "function_name": function[
                        "FunctionName"
                    ],
                    "runtime": function[
                        "Runtime"
                    ]
                })

        # ECS
        ecs = boto3.client(
            "ecs",
            region_name=region
        )

        clusters = ecs.list_clusters()

        for cluster_arn in clusters["clusterArns"]:

            services = ecs.list_services(
                cluster=cluster_arn
            )

            if services["serviceArns"]:

                details = ecs.describe_services(
                    cluster=cluster_arn,
                    services=services["serviceArns"]
                )

                for service in details["services"]:

                    resources["ecs"].append({
                        "region": region,
                        "cluster": cluster_arn.split("/")[-1],
                        "service": service["serviceName"],
                        "running_count": service[
                            "runningCount"
                        ]
                    })

        # ELB
        elb = boto3.client(
            "elbv2",
            region_name=region
        )

        load_balancers = (
            elb.describe_load_balancers()
        )

        for lb in load_balancers["LoadBalancers"]:

            resources["elb"].append({
                "region": region,
                "name": lb["LoadBalancerName"],
                "type": lb["Type"],
                "state": lb["State"]["Code"]
            })

    # S3
    s3 = boto3.client("s3")

    buckets = s3.list_buckets()

    for bucket in buckets["Buckets"]:

        resources["s3"].append({
            "bucket_name": bucket["Name"],
            "created": bucket[
                "CreationDate"
            ].isoformat()
        })

    return resources


def analyze_resources(resources):

    findings = []

    # EBS
    for volume in resources["ebs"]:

        if volume["state"] == "available":

            findings.append({
                "type": "EBS",
                "resource": volume["volume_id"],
                "region": volume["region"],
                "severity": "MEDIUM",
                "finding": "Unattached EBS volume",
                "recommendation": (
                    "Review the volume and delete it "
                    "if it is no longer required."
                )
            })

    # EC2
    for instance in resources["ec2"]:

        cpu = instance.get(
            "cpu_average_7_days"
        )

        if cpu is not None and cpu < 10:

            findings.append({
                "type": "EC2",
                "resource": instance["instance_id"],
                "region": instance["region"],
                "severity": "MEDIUM",
                "finding": (
                    f"Low average CPU utilization: {cpu}%"
                ),
                "recommendation": (
                    "Investigate rightsizing or "
                    "scheduling the instance."
                )
            })

    # NAT Gateway
    for gateway in resources["nat_gateways"]:

        findings.append({
            "type": "NAT Gateway",
            "resource": gateway["nat_gateway_id"],
            "region": gateway["region"],
            "severity": "LOW",
            "finding": "NAT Gateway is active",
            "recommendation": (
                "Review NAT Gateway "
                "data-processing charges."
            )
        })

    return findings


def save_report(result):

    s3 = boto3.client("s3")

    report_key = (
        f"reports/{date.today().isoformat()}/"
        "aws-cost-report.json"
    )

    s3.put_object(
        Bucket=S3_BUCKET,
        Key=report_key,
        Body=json.dumps(
            result,
            indent=2,
            default=str
        ),
        ContentType="application/json"
    )

    return report_key


def save_cost_data(costs):

    s3 = boto3.client("s3")

    lines = ["month,service,cost"]

    for month_data in costs["months"]:

        month = month_data["month"]

        for service_data in month_data["services"]:

            service = service_data["service"]
            cost = service_data["cost"]

            service = service.replace('"', '""')

            lines.append(
                f'"{month}","{service}",{cost}'
            )

    body = "\n".join(lines)

    key = (
        f"cost-data/{date.today().isoformat()}/"
        "cost-history.csv"
    )

    s3.put_object(
        Bucket=S3_BUCKET,
        Key=key,
        Body=body,
        ContentType="text/csv"
    )

    return key


def send_sns_alert(findings):

    if not findings:
        print("No optimization findings. SNS notification not required.")
        return

    sns = boto3.client("sns")

    message_lines = [
        "AWS Cost Optimization Alert",
        "",
        f"Total findings: {len(findings)}",
        ""
    ]

    for finding in findings:

        message_lines.append(
            f"Type: {finding.get('type')}"
        )

        message_lines.append(
            f"Resource: {finding.get('resource')}"
        )

        message_lines.append(
            f"Region: {finding.get('region')}"
        )

        message_lines.append(
            f"Severity: {finding.get('severity')}"
        )

        message_lines.append(
            f"Finding: {finding.get('finding')}"
        )

        message_lines.append(
            f"Recommendation: {finding.get('recommendation')}"
        )

        message_lines.append("")
        message_lines.append("-----------------------------")
        message_lines.append("")

    message = "\n".join(message_lines)

    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="AWS Cost Optimization Alert",
        Message=message
    )

    print("SNS notification sent successfully.")


def lambda_handler(event, context):

    print(
        "Starting AWS Cost Optimization scan..."
    )

    costs = get_costs()

    resources = get_resources()

    findings = analyze_resources(
        resources
    )

    findings.append({
    "type": "TEST",
    "resource": "test-resource",
    "region": "us-east-1",
    "severity": "MEDIUM",
    "finding": "Test optimization finding",
    "recommendation": "This is a test notification. No action required."
})

    send_sns_alert(findings)



    result = {
        "costs": costs,
        "active_resources": resources,
        "optimization_findings": findings
    }

    report_key = save_report(
        result
    )

    cost_data_key = save_cost_data(
        costs
    )

    print(
        f"Report saved to S3: {report_key}"
    )

    print(
        f"Cost data saved to S3: {cost_data_key}"
    )

    return {
        "statusCode": 200,
        "body": json.dumps({
            "report": report_key,
            "cost_data": cost_data_key,
            "optimization_findings": findings
        })
    }