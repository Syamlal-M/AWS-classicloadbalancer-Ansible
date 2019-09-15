# Classic Load Balancer

A load balancer distributes incoming application traffic across multiple EC2 instances in multiple Availability Zones. This increases the 
fault tolerance of your applications. Elastic Load Balancing detects unhealthy instances and routes traffic only to healthy instances.
Here i am creating classic load balancer with two instances.

```
---
 - name: "create the classic load balancer with two instances"
   hosts: localhost
   connection: local
   vars:
     instance_type: t2.micro
     security_group: ansible-sg
     image: ami-0d2692b6acea72ee6
     region: ap-south-1
     keypair: loadbalancer
     exact_count: 1
     subnet: subnet-30820a7c
     subnets:
           - {zone: "ap-south-1b" , subnet: "subnet-30820a7c" , tag: "elb1"}
           - {zone: "ap-south-1a" , subnet: "subnet-5f85de37" , tag: "elb2"}

   tasks:

      - name: "Setup a classic load balancer"
        ec2_elb_lb:
              name: aws-blog-elb
              state: present
              region: ap-south-1
              zones:
                 - ap-south-1a
              listeners:
                - protocol: http
                  load_balancer_port: 80
                  instance_port: 80
        register: awsblog-elb


      - name: "gather facts of elb"
        ec2_elb_facts:
          region: ap-south-1
        register: elb_facts

      - name: " healthcheck for an instances"
        ec2_elb_lb:

          name: aws-blog-elb
          state: present
          region: ap-south-1
          zones:
            - ap-south-1a
          listeners:
            - protocol: http
              load_balancer_port: 80
              instance_port: 80
          health_check:
             ping_protocol: http
             ping_port: 80
             ping_path: "/healthcheck.txt"
             response_timeout: 5
             interval: 10
             unhealthy_threshold: 2
             healthy_threshold: 5


      - name: "debug the security group id"
        debug: var=elb_facts.elbs.0.security_groups


      - name: Creating a security group

        ec2_group:
         name: "{{ security_group }}"
         description: Ansible-webserver
         region: "{{ region }}"
         rules:
           - proto: tcp
             from_port: 22
             to_port: 22
             cidr_ip: 0.0.0.0/0
           - proto: tcp
             from_port: 80
             to_port: 80
             group_id: "{{elb_facts.elbs.0.security_groups}}"
         rules_egress:
           - proto: all
             cidr_ip: 0.0.0.0/0

      - name: "Launching ec2 Instance"
        with_items: "{{subnets}}"
        ec2:
         instance_type: "{{instance_type}}"
         key_name: "{{keypair}}"
         image: "{{image}}"
         user_data: "{{ lookup('file', 'userdata.sh') }}"
         region: "{{region}}"
         group: "{{security_group}}"
         vpc_subnet_id: "{{item.subnet}}"
         zone: "{{item.zone}}"
         wait: yes
         count_tag:
           Name: Ansible-lb-{{item.tag}}
         instance_tags:
           Name: Ansible-lb-{{item.tag}}
         exact_count: "{{exact_count}}"
        register: ec2_status

      - name: "public ip address of first and second instance"
        debug:
         msg:  " {{item}}"
        with_items:

             -  "{{ec2_status.results.0.tagged_instances.0.public_ip}}"
             -  "{{ec2_status.results.1.tagged_instances.0.public_ip}}"

      - name: "Add each EC2 instance to the ELB"
        ec2_elb:
         state: present
         ec2_elbs: aws-blog-elb
         region: "{{ item.region }}"
         instance_id: "{{ item.id }}"
        with_items:
              -  "{{ec2_status.results.0.tagged_instances.0}}"
              -  "{{ec2_status.results.1.tagged_instances.0}}"
