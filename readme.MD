

### **Part 1: Launch an EC2 Instance**
1. **Log in to AWS Management Console**:
   - Navigate to the **EC2 Dashboard**.

2. **Launch a New EC2 Instance**:
   - Select **Ubuntu Server 22.04 LTS** as the operating system.
   - Choose the **t2.micro** or **t2.nano** instance type.
   - Configure the instance with default settings and assign a **key pair** for SSH access.
   - Ensure the instance is in a public subnet and has a **public IP**.

3. **Configure Security Group**:
   - Allow **UDP port 1194** for OpenVPN:
     - **Type**: Custom UDP Rule
     - **Port Range**: 1194
     - **Source**: Anywhere (`0.0.0.0/0`).
   - Allow **SSH (port 22)** for administration:
     - **Type**: SSH
     - **Port Range**: 22
     - **Source**: Your IP (`<your-IP-address>/32`).

---

### **Part 2: Connect to the Instance**
1. Use **EC2 Instance Connect** or an SSH client (like `Terminal` on macOS):
   ```bash
   ssh -i your-key.pem ubuntu@<server-public-ip>
   ```

---

### **Part 3: Install OpenVPN and Easy-RSA**
1. **Update the System**:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install Required Packages**:
   ```bash
   sudo apt install openvpn easy-rsa iptables-persistent -y
   ```

3. **Set Up Easy-RSA**:
   ```bash
   make-cadir ~/easy-rsa
   cd ~/easy-rsa
   ```

---

### **Part 4: Configure Certificates**
1. **Initialize PKI**:
   ```bash
   ./easyrsa init-pki
   ```

2. **Build the Certificate Authority (CA)**:
   ```bash
   ./easyrsa build-ca nopass
   ```
   - Follow the prompts and set a strong passphrase for your CA.

3. **Generate the Server Certificate**:
   ```bash
   ./easyrsa build-server-full server nopass
   ```

4. **Generate the Client Certificate**:
   ```bash
   ./easyrsa build-client-full client1 nopass
   ```

5. **Generate Diffie-Hellman Parameters**:
   ```bash
   ./easyrsa gen-dh
   ```

6. **Generate the TLS Key**:
   ```bash
   sudo openvpn --genkey secret ta.key
   ```

---

### **Part 5: Configure the OpenVPN Server**
1. **Copy Certificates and Keys**:
   ```bash
   sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/
   sudo cp ~/easy-rsa/pki/private/server.key /etc/openvpn/
   sudo cp ~/easy-rsa/pki/issued/server.crt /etc/openvpn/
   sudo cp ~/easy-rsa/ta.key /etc/openvpn/
   sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/
   ```

2. **Create the OpenVPN Configuration File**:
   ```bash
   sudo nano /etc/openvpn/server.conf
   ```
   Add the following:
   ```
   port 1194
   proto udp
   dev tun
   ca /etc/openvpn/ca.crt
   cert /etc/openvpn/server.crt
   key /etc/openvpn/server.key
   dh /etc/openvpn/dh.pem
   auth SHA256
   tls-auth /etc/openvpn/ta.key 0
   cipher AES-256-CBC
   keepalive 10 120
   persist-key
   persist-tun
   push "redirect-gateway def1 bypass-dhcp"
   push "dhcp-option DNS 8.8.8.8"
   push "dhcp-option DNS 8.8.4.4"
   user nobody
   group nogroup
   status /var/log/openvpn-status.log
   log-append /var/log/openvpn.log
   verb 3
   tls-server
   
   # Define the VPN subnet for clients
   server 10.8.0.0 255.255.255.0
   
   # Enable IP forwarding for the VPN
   push "route 10.8.0.0 255.255.255.0"
   
   # Log additional information for debugging
   explicit-exit-notify 1
   ```

3. **Enable IP Forwarding**:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
   ```

4. **Set Up NAT Rules**:
   ```bash
   sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o enX0 -j MASQUERADE
   sudo netfilter-persistent save
   ```

5. **Start OpenVPN**:
   ```bash
   sudo systemctl start openvpn@server
   sudo systemctl enable openvpn@server
   ```

---

### **Part 6: Configure the Client**
1. **Generate the Client Configuration File**:
   ```bash
   nano ~/client1.ovpn
   ```
   Add the following:
   ```plaintext
   client
   dev tun
   proto udp
   remote <server-public-ip> 1194
   resolv-retry infinite
   nobind
   persist-key
   persist-tun
   remote-cert-tls server
   auth SHA256
   cipher AES-256-CBC
   key-direction 1
   verb 3

   <ca>
   (Insert the contents of /etc/openvpn/ca.crt here)
   </ca>
   <cert>
   (Insert the contents of ~/easy-rsa/pki/issued/client1.crt here)
   </cert>
   <key>
   (Insert the contents of ~/easy-rsa/pki/private/client1.key here)
   </key>
   <tls-auth>
   (Insert the contents of ~/easy-rsa/ta.key here)
   </tls-auth>
   ```

2. **Transfer the `.ovpn` File to Your Device**:
   - Use `scp` to copy the file to your Mac or local device:
     ```bash
     scp -i your-key.pem ubuntu@<server-public-ip>:~/client1.ovpn ~/Desktop/
     ```

3. **Import the File**:
   - On your client device (e.g., macOS or mobile), use OpenVPN clients like **Tunnelblick** or **OpenVPN Connect**.
   - Import the `client1.ovpn` file.

4. **Connect to the VPN**:
   - Launch the OpenVPN client and connect.
   - Test your connection by visiting [whatismyip.com](https://www.whatismyip.com/) to ensure your public IP matches the VPN server.

---

### **Part 7: Optimization**
1. **Minimize Costs**:
   - Stop the instance when not in use to save costs:
     ```bash
     aws ec2 stop-instances --instance-ids <your-instance-id>
     ```
   - Use a smaller instance type like `t2.nano` for lower costs.

2. **Monitor Usage**:
   - Use AWS CloudWatch or the billing dashboard to monitor usage and charges.

