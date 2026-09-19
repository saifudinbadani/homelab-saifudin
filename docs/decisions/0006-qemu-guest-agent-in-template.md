Made the decision to install qemu guest agent in template so every child carries it. 
Did this because Terraform will require it because of reporting of IP address with API.
Now template has this package and can be verified from host-side by running 'qm guest cmd'.