   provider "azurerm" {
    features {}
  
    client_id       = "c6483d2d-ac16-4045-b650-cb8d473b7294"
    client_secret   = "yiP8Q~PCsKb6kggtMpn5s.WXZU95mlVVzG1yjbda"
    subscription_id = "bce8d084-cc86-445d-9aca-c197fe404b4e"
    tenant_id       = "e648628f-f65c-40cc-8a28-c601daf26a89"
  }
  
  # Define the virtual network
resource "azurerm_virtual_network" "vnet" {
  name                = "myVNet"
  address_space       = ["10.0.0.0/16"]
  location            = "Central India"
  resource_group_name = "CD-TestBed"
}

# Define a subnet within the virtual network
resource "azurerm_subnet" "subnet" {
  name                 = "mySubnet"
  resource_group_name  = "CD-TestBed"
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Define a public IP for the jump server
resource "azurerm_public_ip" "jump_server_public_ip" {
  name                = "JumpServerPublicIP"
  location            = "Central India"
  resource_group_name = "CD-TestBed"
  allocation_method   = "Dynamic"
}

# Define a network interface for the jump server
resource "azurerm_network_interface" "jump_server_nic" {
  name                = "JumpServerNIC"
  location            = "Central India"
  resource_group_name = "CD-TestBed"

  ip_configuration {
    name                          = "jump-server-ip-config"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id           = azurerm_public_ip.jump_server_public_ip.id
  }
}

# Define the jump server virtual machine
resource "azurerm_virtual_machine" "jump_server" {
  name                  = "JumpServer"
  location              = "Central India"
  resource_group_name   = "CD-TestBed"
  network_interface_ids = [azurerm_network_interface.jump_server_nic.id]
  vm_size               = "Standard_B1s"

  storage_os_disk {
    name              = "Jumpdisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Premium_LRS"
  }

  storage_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }

  os_profile {
    computer_name  = "JumpServer"
    admin_username = "jump"
  }

  os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
      path     = "/home/jump/.ssh/authorized_keys"
      key_data = file("~/.ssh/id_rsa.pub")
    }
  }

}



# Define an availability set for web servers
resource "azurerm_availability_set" "web_servers_avset" {
  name                = "WebServerAvailabilitySet"
  location            = "Central India"
  resource_group_name = "CD-TestBed"
}

# Define network interfaces for web servers
resource "azurerm_network_interface" "web_server_nics" {
  count               = 2
  name                = "WebServerNIC-${count.index}"
  location            = "Central India"
  resource_group_name = "CD-TestBed"

  ip_configuration {
    name                          = "web-server-ip-config"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

# Define the web servers
resource "azurerm_virtual_machine" "web_servers" {
  count                 = 2
  name                  = "WebServer-${count.index}"
  location              = "Central India"
  resource_group_name   = "CD-TestBed"
  availability_set_id   = azurerm_availability_set.web_servers_avset.id
  network_interface_ids = [azurerm_network_interface.web_server_nics[count.index].id]
  vm_size               = "Standard_B1s"

  storage_os_disk {
    name              = "osdisk${count.index}"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Premium_LRS"
  }

  storage_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }

  os_profile {
    computer_name  = "WebServer-${count.index}"
    admin_username = "web"
  }

  os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
      path     = "/home/web/.ssh/authorized_keys"
      key_data = file("~/.ssh/id_rsa.pub")
    }
  }
}
####################################
resource "azurerm_network_security_group" "app_nsg" {
  name                = "app-nsg"
  location            = "Central India"
  resource_group_name = "CD-TestBed"

  security_rule {
    name                       = "Allow_HTTP"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow_SSH"
    priority                   = 201
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "nsg_association" {
  subnet_id                 = azurerm_subnet.subnet.id
  network_security_group_id = azurerm_network_security_group.app_nsg.id

  depends_on = [
    azurerm_network_security_group.app_nsg
  ]
}



#####################################
# Define the public IP for the load balancer


resource "azurerm_public_ip" "lb_public_ip" {
  name                = "PublicIPForLB"
  location            = "Central India"
  resource_group_name = "CD-TestBed"
  allocation_method   = "Static"
  sku                 = "Standard" # Specify the SKU here
}

resource "azurerm_lb" "example" {
  name                = "TestLoadBalancer"
  location            = "Central India"
  resource_group_name = "CD-TestBed"

  sku = "Standard" # Specify the SKU here to match the public IP
  frontend_ip_configuration {
    name                 = "PublicIPAddress"
    public_ip_address_id = azurerm_public_ip.lb_public_ip.id
  }
}


# Define the backend address pool for the load balancer
resource "azurerm_lb_backend_address_pool" "example" {
  loadbalancer_id = azurerm_lb.example.id
  name            = "BackEndAddressPool"
}

# Define backend pool addresses
resource "azurerm_lb_backend_address_pool_address" "example1" {
  count                    = 2
  name                     = "example1-${count.index}"
  backend_address_pool_id  = azurerm_lb_backend_address_pool.example.id
  virtual_network_id       = azurerm_virtual_network.vnet.id
  ip_address               = azurerm_network_interface.web_server_nics[count.index].private_ip_address
}

# Define a health probe for the load balancer
resource "azurerm_lb_probe" "example" {
  loadbalancer_id = azurerm_lb.example.id
  name            = "web-running-probe"
  port            = 80
}

# Define a load balancing rule
resource "azurerm_lb_rule" "example" {
  loadbalancer_id                = azurerm_lb.example.id
  name                           = "LBRule"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 80
  frontend_ip_configuration_name = "PublicIPAddress"
  backend_address_pool_ids        =    [ azurerm_lb_backend_address_pool.example.id ] 
  probe_id                       = azurerm_lb_probe.example.id
}



##########

resource "azurerm_sql_server" "app_server_6008089" {
  name                         = "appserver6008089"
  resource_group_name          = "CD-TestBed"
  location                     = "Central India"  
  version             = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = "Azure@123"
}

resource "azurerm_sql_database" "app_db" {
  name                = "appdb"
  resource_group_name = "CD-TestBed"
  location            = "Central India"  
  server_name         = azurerm_sql_server.app_server_6008089.name
   depends_on = [
     azurerm_sql_server.app_server_6008089
   ]
}

resource "azurerm_sql_firewall_rule" "app_server_firewall_rule" {
  name                = "app-server-firewall-rule"
  resource_group_name = "CD-TestBed"
  server_name         = azurerm_sql_server.app_server_6008089.name
  start_ip_address    = "122.177.111.146"
  end_ip_address      = "122.177.111.146"
}


