
# Google Skils: Implementing Cloud Load Balancing for Compute Engine
- https://www.skills.google/course_templates/648

## Resources
- https://docs.cloud.google.com/load-balancing/docs/load-balancing-overview 
- Set up network Load Balancers
- Set up Application Load Balancers
- Use an internal application load balancer

## Additional Resources
- https://docs.cloud.google.com/sdk/gcloud
- https://cloud.google.com/compute/docs/subnetworks

## Tutorial commands

```bash
# set the default region
gcloud config set compute/region europe-west3

# set the default zone
gcloud config set compute/zone europe-west3-a

# create virtual machines
gcloud compute instances create web1 \
  --zone=europe-west3-a \
  --tags=network-lb-tag \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html'

gcloud compute instances create web2 \
  --zone=europe-west3-a \
  --tags=network-lb-tag \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web2</h3>" | tee /var/www/html/index.html'

gcloud compute instances create web3 \
  --zone=europe-west3-a \
  --tags=network-lb-tag \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'

# create firewall
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80

# configure load balancing service
# create the target pool and forwarding rule

# create a static external addresss for load balancer
gcloud compute addresses create network-lb-ip-1 \
  --region europe-west3

# add a legacy HTTP health check resource
gcloud compute http-health-checks create basic-check

# Create the target pool and forwarding rule
# create the target pool and use the health check, which is required for the service to function
gcloud compute target-pools create www-pool \
  --region europe-west3 --http-health-check basic-check

# add the instances to the pool
gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3

# add forwarding rule
gcloud compute forwarding-rules create www-rule \
    --region  europe-west3 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool



# Create an application load balancer
# load balancer template
gcloud compute instance-templates create lb-backend-template \
   --region=europe-west3 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=debian-12 \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'

# create a managed instance group based on template
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=europe-west3-a

# create fw-allow-health-check firewall rule 
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80

# setup a global static external IP address that can reach the load balancer
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global

# view IPv4 address that was reserved
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global

# create a health check for the load balancer
gcloud compute health-checks create http http-basic-check \
  --port 80

# create a backend serivce
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global

# add instance group as the backend to the backend service
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=europe-west3-a \
  --global

# create URL map to route the incoming request to the default backend service
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service

# crete a target HTTP proxy to route request to URL map
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http

# create a global forwarding rule to route incoming requests to the proxy
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80


# Use an Internal Application Load Balancer
gcloud config set compute/region Region
gcloud config set compute/zone Zone

# Create a virtual environment
# install the virtualenv environment
sudo apt-get install -y virtualenv

# build the virtual environment
python3 -m venv venv

# activate the virtual environment
source venv/bin/activate

# Enable gemini code assist in the cloud shell ide
gcloud services enable cloudaicompanion.googleapis.com

# click open editor on the cloud shell tool
# click Cloud Code - No Project in the status bar at the bottom of the screen
# Authorize the plugin if necessary
# verify the google cloud project (Project ID) displays in the Cloud Code status message

# Create a backend managed instance group
touch ~/backend.sh

# click open editor, open in new window

# add the following sript into the editor
````
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/serveprimes.py
import http.server

def is_prime(a): return a!=1 and all(a % i for i in range(2,int(a**0.5)+1))

class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/plain")
    s.end_headers()
    s.wfile.write(bytes(str(is_prime(int(s.path[1:]))).encode('utf-8')))

http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/serveprimes.py >/dev/null 2>&1 &
````
# click file save
# Click the gemini code assist: smart actions icon and select explain this
# in the inline text box of code assist chat, replace and send:

````
As an Application Developer at Cymbal AI, explain the backend.sh startup script to a new team member. This script is used to run a small Python web server written in a Python file serveprimes.py. Provide a detailed breakdown of the script's key components and explain the function of each command.

For the suggested improvements, don't make any changes to the file's content.
````

# create the istance template primecalc
# blueprint for the backend VMs, has --no-address, meaning these backend VMs won't have public internet address
gcloud compute instance-templates create primecalc \
--metadata-from-file startup-script=backend.sh \
--no-address --tags backend --machine-type=e2-medium

# firewall rule to allow traffic on port 80 to reach the backend VMs.
# crucial for the internal application load balancer and health checks to communicate with them
gcloud compute firewall-rules create http --network default --allow=tcp:80 \
--source-ranges IP --target-tags backend

# create the instance group named backend, start off with 3 instances
gcloud compute instance-groups managed create backend \
--size 3 \
--template primecalc \
--zone ZONE

# setup the internal load balancer
# You're creating that single, private VIP entrance for your internal service. It allows other internal applications to reach your "prime number calculator" reliably, without needing to know which specific backend VM is active or available.
# In this task, you set up the Internal Load Balancer and connect it to the instance group you have just created.
# An Internal Load Balancer consists of three main parts:
# Forwarding Rule: This is the actual private IP address that other internal services send requests to. It "forwards" traffic to your backend service.
# Backend Service: This defines how the load balancer distributes traffic to your VM instances. It also includes the health check.
# Health Check: This is a continuous check that monitors the "health" of your backend VMs. The load balancer only sends traffic to machines that are passing their health checks, ensuring your service is always available.

# create a health -> need to make sure the load balancer only sends traffic to healthy instasnces
gcloud compute health-checks create http ilb-health --request-path /2

# create a backend service - prime-service
# this service ties the health check to the instance group
gcloud compute backend-services create prime-service \
--load-balancing-scheme internal --region=REGION \
--protocol tcp --health-checks ilb-health

# add the instance group to the backend service
# connect your backend instance group to the prime-service backend service
# tells the load balancer which machines it should manage
gcloud compute backend-services add-backend prime-service \
--instance-group backend --instance-group-zone=ZONE \
--region=REGION

# create the forwarding rule, named prime-lb with a static IP
gcloud compute forwarding-rules create prime-lb \
--load-balancing-scheme internal \
--ports 80 --network default \
--region=REGION --address IP \
--backend-service prime-service

# Task 5. Create a public-facing web server
# Now you can see how a public-facing application (like a website) can leverage your internal services.
# In this task, you create a public-facing web server that uses the internal "prime number calculator" service (via the internal Application Load Balancer) to display a matrix of prime numbers.
# First, run the following command to create the startup script for this public-facing "frontend" in the home directory:
touch ~/frontend.sh

# add the following script into the editor
````
sudo chmod -R 777 /usr/local/sbin/
sudo cat << EOF > /usr/local/sbin/getprimes.py
import urllib.request
from multiprocessing.dummy import Pool as ThreadPool
import http.server
PREFIX="http://IP/" #HTTP Load Balancer
def get_url(number):
    return urllib.request.urlopen(PREFIX+str(number)).read().decode('utf-8')
class myHandler(http.server.BaseHTTPRequestHandler):
  def do_GET(s):
    s.send_response(200)
    s.send_header("Content-type", "text/html")
    s.end_headers()
    i = int(s.path[1:]) if (len(s.path)>1) else 1
    s.wfile.write("<html><body><table>".encode('utf-8'))
    pool = ThreadPool(10)
    results = pool.map(get_url,range(i,i+100))
    for x in range(0,100):
      if not (x % 10): s.wfile.write("<tr>".encode('utf-8'))
      if results[x]=="True":
        s.wfile.write("<td bgcolor='#00ff00'>".encode('utf-8'))
      else:
        s.wfile.write("<td bgcolor='#ff0000'>".encode('utf-8'))
      s.wfile.write(str(x+i).encode('utf-8')+"</td> ".encode('utf-8'))
      if not ((x+1) % 10): s.wfile.write("</tr>".encode('utf-8'))
    s.wfile.write("</table></body></html>".encode('utf-8'))
http.server.HTTPServer(("",80),myHandler).serve_forever()
EOF
nohup python3 /usr/local/sbin/getprimes.py >/dev/null 2>&1 &
````

# ask gemini code assist to explain the startup script
````
You are an Application Developer at Cymbal AI. A new team member needs help understanding this startup script, which is used to run a public-facing web server written in the Python file getprimes.py. Explain the frontend.sh script in detail. Break down its key components, the commands used, and their function within the script.

For suggested improvements, do not make any changes to the file's content.
````

# create the frontend instance named frontend
gcloud compute instances create frontend --zone=ZONE \
--metadata-from-file startup-script=frontend.sh \
--tags frontend --machine-type=e2-standard-2

# open firewall for the frontend, allow traffice from anywhere on the internet (0.0.0.0/0)
gcloud compute firewall-rules create http2 --network default --allow=tcp:80 \
--source-ranges 0.0.0.0/0 --target-tags frontend

# In the Navigation menu, click Compute Engine > VM instances. Refresh your browser if you don't see the frontend instance.
# Open the External IP for the frontend in your browser:
# You should see a matrix like this, showing all prime numbers, up to 100, in green:
# Try adding a number to the path, like http://your-ip/10000, to see all prime numbers starting from that number.
```
