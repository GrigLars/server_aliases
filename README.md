# Navigating to servers using ansible, jinja2, ssh config, and bash aliases

I have been using ansible, bash, and jinja2 to help me navigate around various subsystems.  There are two ways I accomplish this: via ssh config and use of the .bash_aliases file.  I grew tired of relying on shoddy or incomplete DNS records, servers that were called one thing once, but changed names, or servers that have next to no decritptive hostnames, for example, ```ip-192-168-123-45``` is your backupmail server. 

### Get your inventory via ansible.
There are a lot of ways to get this, depending on which server inventory you're using.  AWS, DigitalOcean, VMware, hardware, or a combination in a inventory hosts file. Each have their own parsing engine, and so I won't cover that here.  But what you need to get is:

1. The hostname or alias you wish to use at the command line
2. The ssh ip address or login domain name
3. The ssh port (if not 22)
4. The ssh username
5. The ssh key (or you'll have to enter in a password)

### Have it dump to an inventory yaml file

In my case, I dump it as part of my vault file and keep it encrypted.  In the file, I have it setup like this:

    vault_ssh_config:
      - Name: aws-boinc1
        Hostname: awsboinc1.example.net
        User: ubuntu
        Port: 222
        Privatekey: ~/.ssh/aws_boinc_key
      - Name: aws-boinc2
        Hostname: awsboinc2.example.net
        User: ubuntu
        Port: 222
        Privatekey: ~/.ssh/aws_boinc_key
      - Name: raspberrypi4b-1
        Hostname: 192.168.4.1
        User: pi
        Port: 22
        Privatekey: ~/.ssh/rpi_key
      - Name: raspberrypi3b-1
        Hostname: 192.168.3.1
        User: pi
        Port: 22
        Privatekey: ~/.ssh/rpi_key 
      [... etc ...]
      
### Parse the yaml file to an .ssh/config with jinja2 in ansible

I then create the ```.ssh/config``` for the bash user with a template file, like ```templates/ssh_config.j2```

    ForwardX11 yes

    {% for server in vault_ssh_config %}
    Host {{ server.Name }}
      Hostname {{ server.Hostname }}
      Port {{ server.Port }}
      User {{ server.User }}
      IdentityFile {{ server.Privatekey }}
    {% endfor %}

    Host *
      Port 22
      User joeuser
      IdentityFile ~/.ssh/juser_id_ed25519

### Parse the yaml file to an aliases file with jinja2 in ansible

There are a few ways you can do this.  First, dump to a file, like ```.bash_server_aliases``` from ```templates/ssh_aliases.j2```

    {% for server in vault_ssh_config %}
    alias {{ server.Name }}='ssh -i {{ server.Privatekey }} -p {{ server.Port }} {{ server.User }}@{{ server.Hostname }}'
    {% endfor %}

Note: If you already have the server.Privatekey, server.Port, and server.User in your ```.ssh/config``` file, just can just use the "server.Hostname"

Then you can include it in your ```.bashrc``` file:

    if [ -f ~/.bash_server_aliases ]; then
        . ~/.bash_server_aliases
    fi

Or include it immediately with a ```source``` command:

    source ~/.bash_server_aliases
