# Generating Crossplane Providers with Ansible and Operator SDK

A POC work to generate Crossplane provider with Ansible and [Operator SDK](https://github.com/operator-framework/operator-sdk). This might enable auto generating Crossplane managed resources for a wide range of [ansible modules](https://docs.ansible.com/ansible/2.9/modules/list_of_all_modules.html) available. 

## How

This repository contains one resource, `GCP KMS KeyRing`, and is generated with the following steps: 

1. Scaffold `KeyRing` resource using [ansible plugin](https://sdk.operatorframework.io/docs/building-operators/ansible) of operator-sdk.

	```
	operator-sdk init --plugins=ansible --domain=ansible-provider-gcp.crossplane.io
	operator-sdk create api --group kms --version v1alpha1 --kind KeyRing --generate-role 
	```

1. Update the file at `roles/keyring/tasks/main.yml` updated with the following content:

	```
	---
	# TODO: Consider packing these Crossplane specific pre steps as a reusable ansible role
	- name: Get the provider config object
	  community.kubernetes.k8s_info:
	    api_version: gcp.crossplane.io/v1beta1
	    kind: ProviderConfig
	    name: "{{ provider_config_ref.name }}"
	  register: provider_config

	- name: Read the provider credentials secret
	  community.kubernetes.k8s_info:
	    api_version: v1
	    kind: Secret
	    name: "{{ provider_config.resources[0].spec.credentials.secretRef.name }}"  
	    namespace: "{{ provider_config.resources[0].spec.credentials.secretRef.namespace }}"    
	  register: provider_creds_secret       
	###  

	# TODO: Consider auto-generating this step per resource
	- name: create the GCP KMS key ring
	  google.cloud.gcp_kms_key_ring:
	    name: "{{ ansible_operator_meta.name }}"
	    location: "{{ for_provider.location }}"
	    project: "{{ provider_config.resources[0].spec.projectID }}"
	    auth_kind: serviceaccount
	    service_account_contents: "{{ provider_creds_secret.resources[0].data['key.json'] | b64decode }}"
	    state: present
	  register: at_provider_status
	###

	# TODO: Consider packing these Crossplane specific pre steps as a reusable ansible role
	# TODO: Also post Crossplane compatible status conditions
	- name: update status
	  operator_sdk.util.k8s_status:
	    api_version: kms.ansible-provider-gcp.crossplane.io/v1alpha1
	    kind: KeyRing
	    name: "{{ ansible_operator_meta.name }}"
	    namespace: "{{ ansible_operator_meta.namespace }}"
	    status:
	      atProvider: "{{ at_provider_status }}"
	###
	```

## Testing

1. Build and deploy operator and CRD:

	```
	export IMG=<some-registry>/<project-name>:<tag>
	make docker-build
	kind load docker-image --name local-dev $IMG
	make install
	make deploy
	```

1. Create provider config for crossplane provider gcp and corresponding secret

	```
	apiVersion: gcp.crossplane.io/v1beta1
	kind: ProviderConfig
	metadata:
	  name: gcp-provider
	spec:
	  credentials:
	    secretRef:
	      key: key.json
	      name: my-gcp-serviceaccount
	      namespace: crossplane-system
	    source: Secret
	  projectID: test-playground
	```

1. Create following custom resource:

	```
	apiVersion: kms.ansible-provider-gcp.crossplane.io/v1alpha1
	kind: KeyRing
	metadata:
	  name: keyring-sample
	spec:
	  forProvider:
	    location: global
	  providerConfigRef:
	    name: gcp-provider
	```

1. Check CR status and respective resource on GCP console:


	```
	apiVersion: kms.ansible-provider-gcp.crossplane.io/v1alpha1
	kind: KeyRing
	metadata:
	  name: keyring-sample
	  namespace: provider-ansible-gcp-system
	spec:
	  forProvider:
	    location: global
	  providerConfigRef:
	    name: gcp-provider
	status:
	  atProvider:
	    changed: false
	    createTime: "2020-12-17T12:30:43.137042029Z"
	    failed: false
	    name: keyring-sample
	  conditions:
	  - ansibleResult:
	      changed: 1
	      completion: 2020-12-17T13:16:02.878821
	      failures: 0
	      ok: 4
	      skipped: 0
	    lastTransitionTime: "2020-12-17T13:15:55Z"
	    message: Awaiting next reconciliation
	    reason: Successful
	    status: "True"
	    type: Running
	```

## Caveats

- Currently, Operator sdk does not support generating CRD with OpenAPI validation: https://sdk.operatorframework.io/docs/building-operators/ansible/reference/advanced_options/#custom-resources-with-openapi-validation 
- Logs are available in operator logs and sometimes posted CR status conditions
