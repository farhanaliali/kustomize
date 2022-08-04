####   Create base template 
Create a base template 

 application/deployment.yml

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	name: svc-api
	namespace: 
	spec:
	replicas: 1
	selector:
		matchLabels:
		app: svc-api
	strategy:
		rollingUpdate:
		maxSurge: 25%
		maxUnavailable: 25%
		type: RollingUpdate
	template:
		metadata:
		labels:
			app: svc-api
		spec:
		containers:
		- image: public.ecr.aws/aws-observability/aws-for-fluent-bit:latest
			resources:
			limits:
				memory: 100Mi
				cpu : "0.2"
			requests:
				memory: 50Mi
				cpu : "0.1"
			name: fluent-bit
			volumeMounts:
			- name: fluentbit-config
			mountPath: /fluent-bit/etc/
			- name: app-logs
			mountPath: /var/log
			env:
			- name: POD_NAME
			valueFrom:
				fieldRef:
				fieldPath: metadata.name
			- name: POD_NAMESPACE
			valueFrom:
				fieldRef:
				fieldPath: metadata.namespace
			- name: REGION
			value: eu-west-1
			- name: LOG_GROUP_NAME
			value: SWF
			- name: LOG_STREAM_NAME
			value: $(POD_NAMESPACE)/$(POD_NAME)
		- env:
			- name: HOST_IP
			valueFrom:
				fieldRef:
				apiVersion: v1
				fieldPath: status.hostIP
			- name: FUNCTION
			value: ml
			- name: REGION
			value: eu-west-1
			- name: STAGE
			value: default
			- name: APPLICATION_NAME
			value: test-svc
			- name: CONSUL_END_POINT
			value: "http://$(HOST_IP):8500/v1"
			- name: WORKFLOW_NAME
			value: WORKFLOW_NAME_VALUE
			- name: WORKFLOW_VERSION
			value: "WORKFLOW_VERSION_VALUE"
			image: 123456789012.dkr.ecr.eu-west-1.amazonaws.com/swf:test-service-20220609-v1
			resources:
			limits:
				memory: 2000Mi
				cpu : 1000m
			requests:
				memory: 1024Mi
				cpu : 500m
			imagePullPolicy: Always
			lifecycle:
			postStart:
				exec:
				command:
				- /bin/sh
				- -c
				- update-ca-certificates
			livenessProbe:
			failureThreshold: 10
			initialDelaySeconds: 60
			periodSeconds: 10
			successThreshold: 1
			httpGet:
				scheme: HTTPS
				path: "/api/internal/monitoring/healthchecks"
				port: app-port
			timeoutSeconds: 30
			name: svc-api
			ports:
			- containerPort: 443
			protocol: TCP
			name: app-port
			readinessProbe:
			failureThreshold: 10
			initialDelaySeconds: 5
			periodSeconds: 10
			successThreshold: 1
			httpGet:
				scheme: HTTPS
				path: "/api/internal/monitoring/healthchecks"
				port: app-port
			timeoutSeconds: 1
			volumeMounts:
			- mountPath: /etc/ssl/private/
			name: ssl-mount
			- name: app-logs
			mountPath: /mnt/mesos/sandbox/
		dnsPolicy: ClusterFirst
		restartPolicy: Always
		schedulerName: default-scheduler
		terminationGracePeriodSeconds: 30
		nodeSelector:
			eks.amazonaws.com/nodegroup: prod-ml-eu-swf-app-ng
		volumes:
		- name: fluentbit-config
			configMap:
			name: fluentbit-config
		- name: app-logs
			emptyDir: {}
		- name: ssl-mount
			projected:
			defaultMode: 420
			sources:
			- secret:
				items:
				- key: tls.crt
					path: _wildcard_.service.consul.pem
				- key: tls.key
					path: _wildcard_.service.consul.key
				name: default-cert-tls

Create a service base template

	application/servie.yaml


	apiVersion: v1
	kind: Service
	metadata:
	name: svc-api
	namespace: default
	spec:
	ports:
	- port: 443
		protocol: TCP
		targetPort: 443
	selector:
		app: svc-api
	type: ClusterIP


 Create kustomization.yaml


	apiVersion: kustomize.config.k8s.io/v1beta1
	kind: Kustomization
	resources:
	- service.yaml
	- deployment.yaml



Now We have Base template lets create test and production env



######### Testing ENV

	env/test/kustomization.yaml


		apiVersion: kustomize.config.k8s.io/v1beta1
		kind: Kustomization
		bases:
		- ../../application
		namespace: test
		namePrefix: test-
		patchesStrategicMerge:
		- deploymnet.yaml
		- service.yml
		commonLabels:
				app: test-svc-api


Patch the deployment file 

add the resouce which you want to patch 

	env/test/deployment.yaml

	
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	name: svc-api
	spec:
	selector:
		matchLabels:
		app: test-svc-api
	template:
		metadata:
		labels:
			app: test-svc-api
		spec:
			nodeSelector:
			eks.amazonaws.com/nodegroup: test-ml-eu-swf-app-ng

	
For service 

	env/test/service.yaml

		apiVersion: v1
		kind: Service
		metadata:
		name: svc-api
		spec:
		type: NodePort
		selector:
			app: test-svc-api




Testing env is now setup let try to use kustomize 


######### RUN the Test ENV

	kubectl apply -k env/test    OR      kubectl kustomize env/test | kubectl apply -f -




############### Prod ENV

Now configure prod env same like and test env update the required values 

####### RUN the Prod ENV

	kubectl apply -k env/prod    OR     kubectl kustomize env/test | kubectl apply -f -


