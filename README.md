

######### RUN the Test ENV

	kubectl apply -k env/test    OR      kubectl kustomize env/test | kubectl apply -f -


####### RUN the Prod ENV

	kubectl apply -k env/prod    OR     kubectl kustomize env/test | kubectl apply -f -


