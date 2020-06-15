provider "aws"{
    region = "ap-south-1"
        
}

//create instance

resource "aws_instance" "web" {
  ami           = "ami-052c08d70def0ac62"
  instance_type = "t2.micro"
  key_name = "password"
  security_groups = ["allow"]
  tags = {
    Name = "HelloWorld"
  }
}


// create security group


resource "aws_security_group" "allow_80" {
    name        = "allow"
    description = "Allow 80 inbound traffic"
  

  ingress {
    description = "let ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    
  }

ingress {
    description = "let https"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    
  }

ingress {
    description = "let http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow"
  }
}


// create key pair

resource "tls_private_key" "pass1" {
  algorithm   = "RSA"
  rsa_bits = "4096"
}

resource "local_file" "pass2"{
   content = "${tls_private_key.pass1.private_key_pem}"
   filename = "pass.pem"
}

resource "aws_key_pair" "pass3"{
   key_name = "password"
   public_key = "${tls_private_key.pass1.public_key_openssh}"
}

// create ebs volume 


resource "aws_ebs_volume" "ebs" {
  availability_zone = "aws_instance.web.availability_zone"
  size              = 1
}

// attach ebs to instance

resource "aws_volume_attachment" "ebsatt" {
  device_name = "/dev/sdh"
  volume_id   = "${aws_ebs_volume.ebs.id}"
  instance_id = "${aws_instance.web.id}"
  force_detach = true
}

// connecting instance

connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = "${tls_private_key.password.private_key_pem}"
    host     = "${aws_instance.web.public_ip}"

}

//remote login

provisioner "remote-exec" {
    inline = [
      "sudo yum install httpd php git -y",
      "sudo systemctl restart httpd",
      "sudo systemctl enable httpd"
    ]
}

//create S3 bucket

resource "aws_s3_bucket" "bucket" {
  bucket = "mybucket"
  acl    = "public-read"

  versioning {
    enabled = true
  }
}


//adding file to s3 bucket

resource "aws_s3_bucket_object" "object" {
  bucket = "mybucket"
  key    = "picture"
  source = "C:/Users/Administrator/Downloads/beagle.jpg"
  acl = "public_read"

}

//file copying from github to server


resource "null_resource" "nulldrive"{
depends_on = [
aws_volume_attachment.ebs,
]

connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = "${tls_private_key.password.private_key_pem}"
    host     = "${aws_instance.web.public_ip}"
  }

provisioner "remote-exec" {
    inline = [
   
    "sudo mkfs.ext4 /dev/xvdh",
    "sudo mount /dev/xvdh  /var/www/html",
    "sudo rm -rf /var/www/html*",
    "sudo git clone https://github.com/Nowmicaa/hybridcloud/var/www/html/"

]
}

}



