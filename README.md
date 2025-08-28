# onix-project

The purpose of this repo is to automate the creation of kubernetes cluster using terraform and spin up an isolated AWX instance that reads from bigcloud DB in google cloud.

workflow:

1. user uploads terraform code to repo.
2. upon merging, github actions will create a Release, freezing the code using best practice naming convention
3. github actions will then automatically spin up a local resource to execute terraform
4. ansible will upload an updated AWX's backup DB to bigcloud
5. 
6. the terrarform code will perform a couple of actions:

   build the kubernetes cluster then install AWW
7. Once completed ansible shall configure the instance to read from bigcloud
