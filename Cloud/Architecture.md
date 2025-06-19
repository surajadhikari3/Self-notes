
Cloud native app

Have the 4 pillars

![[Pasted image 20250616114601.png]]
Its an approach of building, deploying and managing the application in the cloud computing environment

Consider 4 things:

1. Microservice
2. Containerized (Docker)
3. Dynamic Orchestration (kubernets)
4. CICD (Fall under the devops) 
   
   It makes the system loosely coupled, Resilent to failure, 
   
   Challenges caused by it are :
Operational complexity --> Knowledge of containerization, orchestration, 

Logging Stack

1. ELK
![[Pasted image 20250616122051.png]]


Centralized Logging(As it collects the logs from the multiple services)

--> Elastic search --> Storage and indexing of the logs 
		 Logstash --> Collect the logs, processes , enrich and stores in the elastic search
		 Kibana --> Visualization (Dashboards, charts, graphs)

2. Promethus with Grafana
	Promethus -> Collects the logs and metrics from microservices, dbs, kubernetes clusters
	Grafana --> Visualization (Dashboards, Charts, Graphs)