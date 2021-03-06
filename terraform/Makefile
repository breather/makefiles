MAKEFLAGS       += --warn-undefined-variables --no-print-directory --jobs=1
SHELL           := bash
.SHELLFLAGS     := -eu -o pipefail -c
.DEFAULT_GOAL   := help
.DELETE_ON_ERROR:
.SUFFIXES:

-include $(PWD)/Makefile.local

#
# Default command definitions
#
TERRAFORM       ?= $(TERRAFORM_CMD)
TERRAFORM_CMD   ?= terraform

#
# Default internal variables
#
ERROR         ?= ""

CONFIG_DIR    ?= config

STATE_BUCKET  ?= terraform
STATE_TABLE   ?= terraform
STATE_PREFIX  ?= workspace
STATE_PROFILE ?= $(AWS_PROFILE)
STATE_REGION  ?= us-east-1
#STATE_KEY    ?= # must be defined in variable or included file

#
# Makefile source
#
MAKEFILE_PATH    ?= $(firstword $(MAKEFILE_LIST))
MAKEFILE_VERSION ?= master
MAKEFILE_SOURCE   = https://raw.githubusercontent.com/breather
MAKEFILE_SOURCE  := $(MAKEFILE_SOURCE)/makefiles/$(MAKEFILE_VERSION)
MAKEFILE_SOURCE  := $(MAKEFILE_SOURCE)/terraform/Makefile

#
# Fallback variables
#
ifdef WORKSPACE
ifneq ($(WORKSPACE),$(shell cat .terraform/environment 2>/dev/null))
$(shell rm -f .terraform/make/profile 2> /dev/null)
$(shell rm -f .terraform/environment 2> /dev/null)
endif
endif

AWS_PROFILE ?= $(shell cat .terraform/make/profile 2>/dev/null || echo)
WORKSPACE   ?= $(shell cat .terraform/environment 2>/dev/null)

ifeq ($(AWS_PROFILE),)
AWS_PROFILE    = $(WORKSPACE)
endif

#
# Terraform variables
#
TF_VAR_aws_profile := $(AWS_PROFILE)
export TF_VAR_aws_profile

#
# Targets
#
.PHONY: help
help:
	@echo Terraform make file
	@echo
	@echo Variables
	@printf "\033[36m%-20s\033[0m %s\n" AWS_PROFILE "aws profile to assume"
	@printf "\033[36m%-20s\033[0m %s\n" "- current" "  $(AWS_PROFILE)"
	@printf "\033[36m%-20s\033[0m %s\n" "- default" "  WORKSPACE"
	@printf "\033[36m%-20s\033[0m %s\n" WORKSPACE "environment to target"
	@printf "\033[36m%-20s\033[0m %s\n" "- current" "  $(WORKSPACE)"
	@printf "\033[36m%-20s\033[0m %s\n" "- usually" "  use1dev or use1prod"
	@printf "\033[36m%-20s\033[0m %s\n" STATE_KEY "terraform state path"
	@printf "\033[36m%-20s\033[0m %s\n" "- current" "  $(STATE_KEY)"
	@echo
	@echo Targets:
	@grep -hE '^[a-zA-Z_\-\.]+(.%)?:.*?## .*$$' $(MAKEFILE_LIST) | \
		sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'
	@if [ -n "$(ERROR)" ]; then \
		echo; \
		echo Error:; \
		echo "  $(ERROR)"; \
		echo; \
	fi

.PHONY: .phony
.phony:

apply: ## Applies a terraform execution plan
apply: | .phony .vars.WORKSPACE
apply: .terraform/make/plan.$(WORKSPACE)
apply:
	$(TERRAFORM) apply \
    -parallelism=80 \
    .terraform/make/plan.$(WORKSPACE)
	@$(MAKE) .clean.plan

check: ## Lints and validates terraform files
check: | .phony
check: format validate

clean: ## Cleans the working environment of temporary files
clean: .phony
	rm -fr .terraform/make

debug.%: ## Prints the output of a variable in the Makefile
debug.%: .phony
	@echo $($*)

destroy: ## Destroys the infratructure managed by terraform
destroy: .phony .terraform/terraform.tfstate .terraform/environment
	$(TERRAFORM) destroy \
    -var-file $(CONFIG_DIR)/default.tfvars \
    -var-file $(CONFIG_DIR)/$(WORKSPACE).tfvars

destroy.%: ## Destroys a single resource managed by terraform
destroy.%: .phony .terraform/terraform.tfstate .terraform/environment refresh
	$(TERRAFORM) destroy \
    -var-file $(CONFIG_DIR)/default.tfvars \
    -var-file $(CONFIG_DIR)/$(WORKSPACE).tfvars \
    -target=$*

format: ## Rewrites config files to canonical format
format: .phony
	$(TERRAFORM_CMD) fmt

init: ## Initializes the terraform state
init: | .phony reset
init: .terraform/make/profile \
      .terraform/terraform.tfstate \
      .terraform/environment

init.%: ## Initializes a workspace
init.%: | .phony
init.%: .terraform/make/profile .terraform/terraform.tfstate
	$(TERRAFORM) workspace new $*

list: ## Lists resource identifiers from the state file
list: | .phony
list: .terraform/terraform.tfstate .terraform/environment
	$(TERRAFORM) state list

output: ## Displays all outputs in the state file
output: | .phony
output: .terraform/terraform.tfstate .terraform/environment
	$(TERRAFORM) output

output.%: ## Displays all outputs in the state file for a module
output.%: | .phony
output.%: .terraform/terraform.tfstate .terraform/environment
	$(TERRAFORM) output -module=$*

plan: ## Generates and shows an execution plan
plan: | .phony .clean.plan
plan: update format validate .terraform/make/plan.$(WORKSPACE)

refresh: ## Update local state file against real resources
refresh: .terraform/terraform.tfstate .terraform/environment .phony
	$(TERRAFORM) refresh \
    -var-file $(CONFIG_DIR)/default.tfvars \
    -var-file $(CONFIG_DIR)/$(WORKSPACE).tfvars

reset: ## Resets terraform
reset: .phony clean
	rm -fr .terraform

show: ## Inspects terraform plan
show: .vars
	@$(TERRAFORM_CMD) show .terraform/make/plan.$(WORKSPACE)

show.%: ## Shows a resource in the state
show.%: .terraform/environment
show.%: .vars
	@$(TERRAFORM) state show $*

taint.%: ## Schedule a resource for update (accepts module=xxx)
taint.%: .terraform/environment
taint.%: .phony
	@if [ -n "$(module)" ]; then \
    $(TERRAFORM) taint -module=$(module) $*; \
  else \
    $(TERRAFORM) taint $*; \
  fi

update: ## Downloads and installs modules for the configuration
update: .phony
	@rm -rf .terraform/modules
	$(TERRAFORM_CMD) get -update=true > /dev/null

validate: ## Validates the terraform files
validate: .terraform/terraform.tfstate
validate: .vars
	$(TERRAFORM_CMD) validate \
	-var-file $(CONFIG_DIR)/default.tfvars \
	-var-file $(CONFIG_DIR)/$(WORKSPACE).tfvars

#
# Non-terraform targets
#
make.update: ## Updates the makefile from git
make.update: .phony
	@wget --unlink -q -O $(MAKEFILE_PATH) $(MAKEFILE_SOURCE)

#
# Interal / Prival targets
#
.vars: # Enforce the present of certain variables
.vars: .vars.WORKSPACE .vars.STATE_KEY

.vars.%: .phony
	@if [ ! -n "$($*)" ]; then \
     echo "Variable $* is not defined"; \
     exit 1; \
  fi

.clean.plan: # Cleans only the plan files
.clean.plan: .phony
	rm -f .terraform/make/plan.*

.terraform/make: # Ensures the presence of the build directory
.terraform/make:
	mkdir -p .terraform/make

.terraform/make/profile: # Ensures the presence of the AWS_PROFILE config
.terraform/make/profile: .terraform/make
	@echo "$(AWS_PROFILE)" > .terraform/make/profile

.terraform/make/plan.%: # Generates a plan file to apply against
.terraform/make/plan.%: .terraform/environment
	$(TERRAFORM) plan \
    -module-depth=-1 \
    -out .terraform/make/plan.$* \
    -parallelism=80 \
    -refresh=true \
    -var-file $(CONFIG_DIR)/default.tfvars \
    -var-file $(CONFIG_DIR)/$*.tfvars

.terraform/environment: # Ensures that a workspace is active
.terraform/environment: | .vars
	$(TERRAFORM) workspace select $(WORKSPACE)

.terraform/terraform.tfstate: # Ensures that remote state was initialized
.terraform/terraform.tfstate: | .vars .terraform/make/profile
	$(TERRAFORM) init \
    -backend-config="acl=private" \
    -backend-config="bucket=$(STATE_BUCKET)" \
    -backend-config="dynamodb_table=$(STATE_TABLE)" \
    -backend-config="profile=$(STATE_PROFILE)" \
    -backend-config="key=$(STATE_KEY)" \
    -backend-config="region=$(STATE_REGION)" \
    -backend-config="workspace_key_prefix=$(STATE_PREFIX)" \
    -force-copy \
    -input=false

#  vim: set ft=make ts=2 sw=2 noet :
