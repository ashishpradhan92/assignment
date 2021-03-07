# assignment
take home assignment cloudformation

The example template contains an Auto Scaling group with a LoadBalancer, a security group that defines ingress rules, CloudWatch alarms, and scaling policies.

The template has three input parameters: InstanceType is the type of EC2 instance to use for the Auto Scaling group and has a default of m1.small; WebServerPort is the TCP port for the web server and has a default of 8888; KeyName is the name of an EC2 key pair to be used for the Auto Scaling group. KeyName must be specified at stack creation (parameters with no default value must be specified at stack creation).

The AWS::AutoScaling::AutoScalingGroup resource WebServerGroup declares the following Auto Scaling group configuration:

1. AvailabilityZones specifies the Availability Zones where the Auto Scaling group's EC2 instances will be created. The Fn::GetAZs function call { "Fn::GetAZs" : "" } specifies    all Availability Zones for the region in which the stack is created.

2. MinSize and MaxSize set the minimum and maximum number of EC2 instances in the Auto Scaling group.

3. LoadBalancerNames lists the LoadBalancers used to route traffic to the Auto Scaling group. The LoadBalancer for this group is the ElasticLoadBalancer resource.

The AWS::AutoScaling::LaunchConfiguration resource LaunchConfig declares the following configurations to use for the EC2 instances in the WebServerGroup Auto Scaling group:

1. KeyName takes the value of the KeyName input parameter as the EC2 key pair to use.

2. UserData is the Base64 encoded value of the WebServerPort parameter, which is passed to an application .

3. SecurityGroups is a list of EC2 security groups that contain the firewall ingress rules for EC2 instances in the Auto Scaling group. In this example, there is only one          security group and it is declared as a AWS::EC2::SecurityGroup resource: InstanceSecurityGroup. This security group contains two ingress rules: 1) a TCP ingress rule that        allows access from all IP addresses ("CidrIp" : "0.0.0.0/0") for port 22 (for SSH access) and 2) a TCP ingress rule that allows access from the ElasticLoadBalancer resource      for the WebServerPort port by specifying the LoadBalancer's source security group. The GetAtt function is used to get the SourceSecurityGroup.OwnerAlias and                      SourceSecurityGroup.GroupName properties from the ElasticLoadBalancer resource. For more information about the Elastic Load Balancing security groups, see Manage security        groups in Amazon EC2-Classic or Manage security groups in Amazon VPC.

4. ImageId is the evaluated value of a set of nested maps. We added the maps so that the template contained the logic for choosing the right image ID. That logic is based on the    instance type that was specified with the InstanceType parameter (AWSInstanceType2Arch maps the instance type to an architecture 32 or 64) and the region where the stack is      created (AWSRegionArch2AMI maps the region and architecture to a image ID):

{ "Fn::FindInMap" : [ "AWSRegionArch2AMI",
   { "Ref" : "AWS::Region" },
   { "Fn::FindInMap" : [ "AWSInstanceType2Arch",
      { "Ref" : "InstanceType" },
      "Arch" ]
   }
   ]}
   
For example, if you use this template to create a stack in the us-east-2 region and specify m1.small as InstanceType, AWS CloudFormation would evaluate the inner map for AWSInstanceType2Arch as the following:
{ "Fn::FindInMap" : [ "AWSInstanceType2Arch", "t2.medium", "Arch" ] }


The AWS::ElasticLoadBalancing::LoadBalancer resource ElasticLoadBalancer declares the following LoadBalancer configuration:

1. AvailabilityZones is a list of Availability Zones where the LoadBalancer will distribute traffic. In this example, the Fn::GetAZs function call { "Fn::GetAZs" : "" }            specifies all Availability Zones for the region in which the stack is created.

2. Listeners is a list of load balancing routing configurations that specify the port that the LoadBalancer accepts requests, the port on the registered EC2 instances where the    LoadBalancer forwards requests, and the protocol used to route requests.

3. HealthCheck is the configuration that Elastic Load Balancing uses to check the health of the EC2 instances that the LoadBalancer routes traffic to. In this example, the HealthCheck targets the root address of the EC2 instances using the port specified by WebServerPort over the HTTP protocol. If the WebServerPort is 8888, the { "Fn::Join" : [ "", ["HTTP:", { "Ref" : "WebServerPort" }, "/"]]} function call is evaluated as the string HTTP:8888/. It also specifies that the EC2 instances have an interval of 30 seconds between health checks (Interval). The Timeout is defined as the length of time Elastic Load Balancing waits for a response from the health check target (5 seconds in this example). After the Timeout period lapses, Elastic Load Balancing marks that EC2 instance's health check as unhealthy. When an EC2 instance fails 5 consecutive health checks (UnhealthyThreshold), Elastic Load Balancing stops routing traffic to that EC2 instance until that instance has 3 consecutive healthy health checks at which point Elastic Load Balancing considers the EC2 instance healthy and begins routing traffic to that instance again.

The AWS::AutoScaling::ScalingPolicy resource WebServerScaleUpPolicy is a simple scaling policy that scales up the Auto Scaling group WebServerGroup. The AdjustmentType property is set to ChangeInCapacity. This means that the ScalingAdjustment represents the number of instances to add (if ScalingAdjustment is positive, instances are added; if negative, instances are deleted). In this example, ScalingAdjustment is 1; therefore, the scaling policy increments the number of EC2 instances in the group by 1 when the policy is executed. The Cooldown property specifies that Amazon EC2 Auto Scaling waits 60 seconds before starting any other scaling activities due to simple scaling policies.

The AWS::CloudWatch::Alarm resource CPUAlarmHigh specifies the simple scaling policy WebServerScaleUpPolicy as the action to execute when the alarm is in an ALARM state (AlarmActions). The alarm monitors the EC2 instances in the WebServerGroup Auto Scaling group (Dimensions). The alarm measures the average (Statistic) EC2 instance CPU utilization (Namespace and MetricName) of the instances in the WebServerGroup (Dimensions) over a 300 second interval (Period). When this value (average CPU utilization over 300 seconds) remains greater than 90 percent (ComparisonOperator and Threshold) for 2 consecutive periods (EvaluationPeriod), the alarm will go into an ALARM state and CloudWatch will execute the WebServerScaleUpPolicy policy (AlarmActions) described above scale up the WebServerGroup.
