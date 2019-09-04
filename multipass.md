# Create a virtual machine locally

## Prerequisites
Install multipass using homebrew:
```
brew cask install multipass
```

## Steps
Create the virtual machine:
```
multipass launch bionic -n leanai -m 8G -d 40G -c 4
```
Enter the virtual machine:
```
multipass shell leanai
```
Install kubernetes:
```
sudo bash

snap install microk8s --classic --beta
```