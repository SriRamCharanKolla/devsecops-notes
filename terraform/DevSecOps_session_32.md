Wednesday, 25 February 2026

Session 32 - For each loop, Dynamic loop, Data sources, Terraform functions.
Notes
resource "type_of_resource" "name_of_resource"{
	key=value
}

variable "name_of_variable" {
	type = string
	default = ""
}

terraform.tfvars
ENV variables TF_VAR_VAR_NAME
-var "var_name=value"

expression ? "true_val" : "false_val"

count.index

list -> ordered,index based values -> allows duplicates
set -> order not gaurenteed, remove duplicates

count based loop -> list

for_each loop
===============
set or map
each -> special variable

count, for_each, dynamic(only for repeated code inside resource)
count -> list based
for_each -> map or set

data sources
============
if you want to get the information from provider, we can use data sources

functions
=============
we can't write custom functions in terraform.

FName
MName
LName

merge
=====
{
	a = "b"
	c = "d"
}

{
	d = "e",
	c = "z"
}

{
	d = "e",
	e = "f"
}

a = "b"
c = "d"
d = "e"
e = "f"


Commands:
terraform output -json > ec2_output.json => To save output in  “ec2_output.json” file.

Timestamps:
Interview question = 
QA = 01:26:28

Mistakes & Learning:

https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049


Doubts Link Clarification AI chat link: 
https://chat.z.ai/c/f3862eee-15cb-49ca-84e8-41702b207049