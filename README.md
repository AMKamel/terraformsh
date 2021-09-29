# Requirements
 - Bash (v3+)
 - Terraform
 - AWS CLI (only for aws_bootstrap command)

# About
  Terraformsh is a Bash script that makes it easier to run Terraform by 
  performing common steps for you. It also makes it easy to keep your
  configuration DRY and deploy infrastructure based on a directory
  hierarchy of environments. See [DRY_CODE.md](./DRY_CODE.md) for
  more details.

  Unlike Terragrunt, this script includes no DSL or code generation. All it
  does is make it easier to call Terraform. See [PHILOSOPHY.md](./PHILOSOPHY.md)
  for more details.

  Terraformsh will detect and use Terraform `-var-files` and `-backend-config`
  configuration files across a directory hierarchy. It also has its own 
  configuration file so you don't have to remember any command other than
  `terraformsh` itself (removing the need for a `Makefile`, though you can
  still use one if you want).

  You can override any options with environment variables, command-line options
  and config files. Good conventions like using .plan files for changes are the
  default.


## How it works

  Change to the directory of a Terraform module and run `terraformsh` with any
  Terraform commands and arguments you'd normally use.

       $ cd rootmodules/aws-infra-region/
       $ terraformsh plan
       $ terraformsh apply

  Terraformsh will run dependent Terraform commands when necessary. If you run
  `terraformsh plan`, Terraformsh will first run `terraform validate`, but before
  that `terraform get`, but before that `terraform init`. Terraformsh passes
  relevant options to each command as necessary, and you can also override
  those options.

  You can even pass multiple Terraform commands as options and it'll run them
  in the order you specify.

  Not sure what that looks like? Use the dry-run mode:

        $ ./terraformsh -N plan apply
        .//terraformsh: WARNING: No -b option passed! Potentially using only local state.

        + terraform init -input=false -reconfigure -force-copy
        + terraform get -update=true
        + terraform validate
        + terraform plan -input=false -out=/home/vagrant/git/PUBLIC/terraformsh/tf.104900abc1.plan
        + terraform init -input=false -reconfigure -force-copy
        + terraform apply -input=false /home/vagrant/git/PUBLIC/terraformsh/tf.104900abc1.plan


  You can tell Terraformsh to change to a module's directory before running commands
  so you don't have to do it yourself (later versions of Terraform have an option
  for this, but earlier ones don't):

        $ ./terraformsh -C ../../../root-modules/aws/common/ plan


  You can pass Terraform configuration files using the `-f` or `-b` options. And
  to make this even simpler, if you pass any argument to Terraformsh after the 
  initial *OPTIONS*, and they match *TFVARS* ('\*.backend.tfvars', '\*.backend.sh.tfvars',
  '\*.tfvars.json', '\*.tfvars', '\*.sh.tfvars.json', '\*.sh.tfvars'), they will
  be loaded with the `-f` and `-b` options automatically.

        $ terraformsh -C ../../.../root-modules/aws/common/ \
            -f terraform.tfvars.json \
            -f override.auto.tfvars.json \
            -b backend.tfvars \
            -b backend-key.tfvars \
            plan approve apply


  Finally, if in any *parent directory* of where you ran Terraformsh, thee are
  files named `backend.sh.tfvars`, `terraform.sh.tfvars.json`, or `terraform.sh.tfvars`,
  those will also be loaded automatically (you can disable this with the `-I` option).


### Environment Variables / Configuration

  Don't want to remember what options to pass to terraformsh? You don't have to!
  You can capture anything you want Terraformsh to do in a config file that is
  automatically loaded.

  You can override the following default variables with environment variables, or
  set them in a bash configuration file (`/etc/terraformsh`, `~/.terraformshrc`,
  `.terraformshrc`, `terraformsh.conf`), with the following key=value pairs:

    DEBUG=0
    TERRAFORM=terraform
    TF_PLANFILE=terraform.plan
    TF_DESTROY_PLANFILE=terraform-destroy.plan
    TF_BOOTSTAP_PLANFILE=terraform-bootstrap.plan
    USE_PLANFILE=1
    INHERIT_TFFILES=1
    NO_DEP_CMDS=0
    CD_DIR=             # The directory to change to before running terraform commands

  The environment variable *TF_DATA_DIR* is automatically overridden by Terraformsh,
  as it uses a new temporary directory for the data dir, based on *both* the name of
  the directory you ran Terraformsh from, and the name of the Terraform module
  you run terraform against (the `-C` option). If you pass your own *TF_DATA_DIR*
  environment variable, Terraformsh will just use that.

  The following can be set in the Terraformsh config file as Bash arrays, or you
  can set them by passing them to `-E`, such as `-E "CD_DIRS=(../some-dir/)"`.

    VARFILES=()			# files to pass to -var-file
    BACKENDVARFILES=() 		# files to pass to -backend-config
    CMDS=()			# the commands for terraformsh to run
    PLAN_ARGS=(-input=false)	# the arguments for 'terraform plan'
    APPLY_ARGS=(-input=false)	# the arguments for 'terraform apply'
    PLANDESTROY_ARGS=(-input=false) # arguments for 'plan -destroy'
    DESTROY_ARGS=(-input=false)	# arguments for 'terraform destroy'
    REFRESH_ARGS=(-input=false)	# arguments for 'terraform refresh'
    INIT_ARGS=(-input=false)	# arguments for 'terraform init'
    IMPORT_ARGS=(-input=false)	# arguments for 'terraform import'
    GET_ARGS=(-update=true)	# arguments for 'terraform get'
    STATE_ARGS=()		# arguments for 'terraform state'

  To use the 'aws_bootstrap' command, pass the '-b FILE' option and make sure the
  file(s) have the following variables:

    bucket          - The S3 bucket your Terraform state will live in
    dynamodb_table  - The DynamoDB table your Terraform state will be managed in

### More Examples

  There are many ways to use Terraformsh, whether you pass all the options
  via environment variables/command-line options, or keep all the commands
  in a configuration file and load everything automatically.

 - Run 'plan' using a `.terraformshrc` file that has all the above options,
   but override terraformsh's internal arguments to 'terraform plan':

        $ terraformsh -E 'PLAN_ARGS=("-compact-warnings" "-no-color" "-input=false")' \
          plan

 - Run 'plan' on a module and pass any configs found in these directories:

        $ terraformsh -C rootmodules/my-database/ \
           *.tfvars  *.backend.tfvars \
           env/my-database/*.tfvars  env/my-database/*.backend.tfvars \
           plan

 - Run 'plan' on a module, implicitly loading configuration files from parent directories:

        $ pwd
        /home/vagrant/git/some-repo/env/non-prod/us-east-2/my-database
        $ echo 'CD_DIRS=(../../../../modules/my-database/)' > terraformsh.conf
        $ echo 'aws_account_id = "0123456789"' > ../../terraform.sh.tfvars
        $ echo 'region = "us-east-2"' > ../terraform.sh.tfvars
        $ echo 'database_name = "some database"' > terraform.sh.tfvars
        $ terraformsh plan

---


    terraformsh v0.8
    Usage: ./terraformsh [OPTIONS] [TFVARS] COMMAND [..]

# Options

  Pass these *OPTIONS* before any others (see examples); do not pass them after
  *TFVARS* or *COMMAND*s.

    -f FILE         A file passed to Terraform's -var-file option.
                    (config: VARFILES=)
    -b FILE         A file passed to Terraform's -backend-config option.
                    (config: BACKENDVARFILES=)
    -C DIR          Change to directory DIR. (config: CD_DIR=)
    -c file         Specify a '.terraformshrc' configuration file to load
    -E EXPR         Evaluate an expression in bash ('eval EXPR').
    -I              Disables automatically loading any 'terraform.sh.tfvars',
                    'terraform.sh.tfvars.json', or 'backend.sh.tfvars' files 
                    found while recursively searching parent directories.
                    (config: INHERIT_TFFILES=0)
    -P              Do not use '.plan' files for plan/apply/destroy commands.
                    (config: USE_PLANFILE=0)
    -D              Don't run 'dependency' commands (e.g. don't run "terraform
                    init" before "terraform apply"). (config: NO_DEP_CMDS=1)
    -N              Dry-run mode (don't execute anything). (config: DRYRUN=1)
    -v              Verbose mode. (config: DEBUG=1)
    -h              This help screen

# Commands

  The following are commands that terraformsh provides wrappers for. Commands
  not listed here will be passed to terraform verbatim, along with any options.

    plan              Run init, get, validate, and `terraform plan -out $TF_PLANFILE`
    apply             Run init, get, validate, and `terraform apply $TF_PLANFILE`
    plan_destroy      Run init, get, validate, and `terraform plan -destroy -out=$TF_DESTROY_PLANFILE`
    destroy           Run init, get, validate, and `terraform apply $TF_DESTROY_PLANFILE`
    shell             Run init, get, and `bash -i -l`
    refresh           Run init, and `terraform refresh`
    validate          Run init, get, and `terraform validate`
    init              Run clean_modules, and `terraform init`
    clean             Remove '.terraform/modules/*', terraform.tfstate files, and .plan files
    clean_modules     Run `rm -v -rf .terraform/modules/*`
    approve           Prompts the user to approve the next step, or the program will exit with an error.
    aws_bootstrap     Looks for 'bucket' and 'dynamodb_table' in your '-b' file options.
                      If found, creates the bucket and table and initializes your Terraform state with them.
    import            Run `terraform import [...]`
    state             RUn `terraform state [...]`
