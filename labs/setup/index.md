# Lab Setup 
## Cloud9
In the Cloud9 terminal download the `lab.pem` using the command from the spreadsheet. 

### Set permission on SSH key 
```
chmod 600 /path/to/lab.pem
```

### SSH to lab servers 
The username for SSH is `ubuntu`
```
ssh -i /path/to/lab.pem ubuntu@<LAB IP> 
```
