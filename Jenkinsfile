pipeline {
  agent any
  environment {
    ISO_FILENAME = 'ubuntu-16.04.3-server-amd64.iso'  
  }
  parameters {
  	string(name: 'FULLNAME', defaultValue: 'Ubuntu', description: 'Full Name')
  	string(name: 'USERNAME', defaultValue: 'ubuntu', description: 'Username')
    string(name: 'HOSTNAME', defaultValue: 'unassigned-hostname', description: 'Hostname')
    string(name: 'DOMAIN', defaultValue: 'unassigned-domain', description: 'Domain')
    string(name: 'TIMEZONE', defaultValue: 'US/Eastern', description: 'Timezone')
    string(name: 'ROOT_DEV', defaultValue: '/dev/sda', description: 'System disk')
  }
  stages {
    stage('Download ISO') {
      when {
        expression {
          return !(fileExists("$ISO_FILENAME"))
        }
        
      }
      steps {
        sh "curl -O http://releases.ubuntu.com/16.04/$ISO_FILENAME"
      }
    }
    stage('Mount ISO') {
      steps {   
        dir('iso') {
          deleteDir()
        }         
        sh '7z x ./$ISO_FILENAME -oiso'
      }
    }
    stage('Update initrd') {
	  steps {
	    dir('initrd') {
	      sh 'cat ../iso/install/initrd.gz | gzip -d > "./initrd"'
	      sh 'find "../custom" | fakeroot cpio -o -H newc -A -F "./initrd"'
	      sh 'cat "./initrd" | gzip -9c > "../iso/install/initrd.gz"'
	      deleteDir()
	    }
	  }    
    }
    stage('Configure') {
      steps {
        sh '''
        PASSWORD=$(openssl rand -base64 32)
        echo "Password for ${USERNAME} is: \'${PASSWORD}\', make sure to change it on first boot"
        set PASSWORD_CRYPT=$(mkpasswd -m sha-512 -S $(pwgen -ns 16 1) ${PASSWORD})
        '''      
      	sh 'echo en > ./iso/isolinux/lang'
      	sh 'sed -i -r "s#timeout\\s+[0-9]+#timeout 10#g" ./iso/isolinux/isolinux.cfg'
        sh 'cp ./preseed/server.seed ./iso/preseed'
        sh 'sed -i "s/FULLNAME/${FULLNAME}/g" ./iso/preseed/server.seed'
        sh 'sed -i "s/USERNAME/${USERNAME}/g" ./iso/preseed/server.seed'
        sh 'sed -i "s/PASSWORD/${PASSWORD_CRYPT}/g" ./iso/preseed/server.seed'
        sh 'sed -i "s/HOSTNAME/${HOSTNAME}/g" ./iso/preseed/server.seed'
        sh 'sed -i "s/DOMAIN/${DOMAIN}/g" ./iso/preseed/server.seed'
        sh 'sed -i "s#ROOT_DEV#${ROOT_DEV}#g" ./iso/preseed/server.seed'
        sh 'sed -i "s#TIMEZONE#${TIMEZONE}#g" ./iso/preseed/server.seed'
        sh '''        	
			MD5_SUM=$(md5sum ./iso/preseed/server.seed)
			sed -i "/label install/ilabel autoinstall\\n\\
			  menu label ^Autoinstall Ubuntu 16.04 Server\\n\\
			  kernel /install/vmlinuz\\n\\
			  append file=/cdrom/preseed/ubuntu-server.seed initrd=/install/initrd.gz auto=true priority=high preseed/file=/cdrom/preseed/server.seed preseed/file/checksum=$MD5_SUM --" ./iso/isolinux/txt.cfg
        '''
      }
    }
    stage('Make ISO') {
      steps {      
        sh 'mkisofs -D -r -V "Unattended Ubuntu Server 16.04" -cache-inodes -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ${ISO_FILENAME}_unattend.iso ./iso'
      }
    }   
  }
  post {
    success {
      archive "${ISO_FILENAME}_unattend.iso"
    }
	  always {
	    dir('iso') {
          deleteDir()
      }
	  }
	}   
}