cd testSecureLxc
docker-compose down
docker-compose build --no-cache
docker-compose up -d

docker exec -it secure-ubuntu-lab /bin/bash
ssh -p 2222 secureuser@localhost    (to rm old hanshake: ssh-keygen -R "[localhost]:2222")
sudo fail2ban-client status
sudo fail2ban-client status sshd