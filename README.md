# Recursive DNS Server and Benchmark
How to install your own recursive DNS, and benchmarks made to a few popular recursive DNS servers

---

<b>INTRODUCTION</b>

In summary, DNS is a service that translates domain names to IP addresses, so for example when you try to browse to google.com the service translates that name (domain) to its internet address (IP)

If you somehow improve your DNS speed you have an overall better quality of experience, specially browsing the web

There are several public DNS servers to choose from

But in theory, if you run your own local recursive DNS, you would have a faster DNS lookup than having to wait for an external answer

There is also the benefit of reliability: If somehow a public DNS has problems, you wouldn't be affected 

To measure the potential benefits, I decided to test and benchmark popular recursive DNS applications against external DNS servers


---

<b>THE BENCHMARK</b>

The idea of the benchmark is to answer a few questions:

If it is faster to use a local recursive DNS, it is how much faster?
In theory if you compile an application it runs faster than installing the prepackaged binary, in this context it would matter?
Will one of the applications tested be faster than the others, or they are more or less equivalent?


---

<b>THE ENVIRONMENT</B>

I deployed virtual machines running Debian 10 with 1gb ram, 4 cpu cores, each one running one of the following packages
Unbound 1.9.0 From Repository
Unbound 1.13.1 Compiled from source
PowerDNS 4.1.11 From Repository
PowerDNS 4.4.3 Compiled from source
Windows Server 2016 DNS

The configuration was left to the defaults, or the minimum necessary to work

And tested them against Google (8.8.8.8), CloudFlare (1.1.1.1), Quad9 (9.9.9.9), OpenDNS (208.67.222.222) and Freenom (80.80.80.80) Public DNS Servers

The client used to test was another Debian 10 with dnseval installed

The command line for dnseval used was: <b>dnseval -f servers.txt -c 100 google.com</b>

The servers.txt file containing the IP addresses of the tested servers (The local virtual machines plus the external public dns servers)


---

<b>RESULTS</b>

The results where as follows:

    server                   avg(ms)     min(ms)     max(ms)     stddev(ms)  lost(%)  ttl        flags                  response
    Unbound Package          0.601       0.226       1.676       0.291       %0       103        QR -- -- RD RA -- --   NOERROR             
    Unbound Compiled         0.574       0.210       2.184       0.296       %0       103        QR -- -- RD RA -- --   NOERROR             
    PowerDNS Package         0.673       0.267       5.161       0.568       %0       104        QR -- -- RD RA -- --   NOERROR             
    PowerDNS Compiled        0.584       0.306       2.067       0.284       %0       104        QR -- -- RD RA -- --   NOERROR             
    Windows 2016 Server DNS  0.569       0.299       1.299       0.256       %0       185        QR -- -- RD RA -- --   NOERROR             
    Cloudflare DNS           4.275       2.725       25.837      2.738       %0       77         QR -- -- RD RA -- --   NOERROR             
    Google DNS               5.399       2.327       134.285     13.493      %0       207        QR -- -- RD RA -- --   NOERROR             
    Quad9 DNS                112.024     107.886     159.600     7.465       %0       15         QR -- -- RD RA -- --   NOERROR             
    OpenDNS DNS              50.535      46.437      61.969      3.455       %0       300        QR -- -- RD RA -- --   NOERROR             
    Freenom DNS              161.385     160.021     170.726     1.625       %0       138        QR -- -- RD RA -- --   NOERROR


---

<b>CONCLUSION</b>

By analyzing the results we can determine:
*It is indeed faster to use a local recursive DNS
*There is no noticeable difference between prepackaged binaries and compiled application in this context
*Cloudfare and Google, being at an average of 4-5 milliseconds would to the end user be no different to the local DNS
*OpenDNS would be middle ground
*Quad9 and Freenom would make a difference to the end user perception, anything above 100ms is quite perceptible
*There is no noticeable benefit in compiling the application, other than having the most recent version of it

So deploying a recursive DNS in your network is a good investment of time and resources to improve the overall quality of experience for end users

Even better if you intercept/redirect DNS requests on the firewall to your recursive DNS server, that way any application that is hardcoded to access an external DNS instead of relying on the local DNS will end up using your local DNS anyway

This is easily done with a NAT rule on LAN intercepting ports 53, 853 and 5353 TCP & UDP to your local DNS IP, and you can check by adding manually one public DNS on your computer and testing with dnsleaktest.com, if the results shows your external IP address instead of the public DNS address, it worked

Bellow are the steps used to install and configure the applications used in this benchmark, except the Windows DNS server, which is so easy to install it does not warrant a tutorial.

---

<b>UNBOUND PACKAGED</b>

    sudo apt install unbound
    
    sudo nano /etc/unbound/unbound.conf
    
    server:
        interface: 0.0.0.0
        access-control: 172.16.0.0/12 allow
    	
    sudo service unbound restart    


---

<b>UNBOUND COMPILED</b>

    sudo apt build-dep unbound
    
    sudo apt install wget
    
    wget https://www.nlnetlabs.nl/downloads/unbound/unbound-1.13.1.tar.gz
    
    tar -xvf *.gz && cd unbound-1.13.1
    
    sudo ./configure --enable-systemd
    
    sudo make -j$(nproc)
    
    sudo make install
    
    sudo ln contrib/unbound.service /usr/lib/systemd/system
    
    sudo useradd unbound
    
    sudo nano /usr/local/etc/unbound/unbound.conf
    
    server:
        interface: 0.0.0.0
        access-control: 172.16.0.0/12 allow
    
    sudo systemctl enable unbound
    
    sudo service unbound start


---

<b>POWERDNS PACKAGED</b>

    sudo apt install pdns-recursor
    
    sudo nano /etc/powerdns/recursor.conf
    
    allow-from=172.16.0.0/12
    local-address=0.0.0.0
    
    sudo service pdns-recursor restart


---

<b>POWERDNS COMPILED</b>

Note: PowerDNS didn't compile with only 1gb ram on the virtual machine (out of memory errors), I had to raise to 2gb, compile, and return it to 1gb for testing

    sudo apt build-dep pdns-recursor
    
    sudo apt install wget
    
    wget https://downloads.powerdns.com/releases/pdns-recursor-4.4.3.tar.bz2
    
    tar -xvf *.bz2 && cd pdns-recursor-4.4.3
    
    sudo mkdir /etc/powerdns
    
    sudo ./configure --with-systemd --sysconfdir=/etc/powerdns
    
    sudo make -j$(nproc)
    
    sudo make install
    
    sudo useradd pdns-recursor
    
    sudo cp pdns-recursor.service /etc/systemd/system
    
    sudo systemctl enable pdns-recursor
    
    sudo nano /etc/powerdns/recursor.conf
    
    allow-from=172.16.0.0/12
    local-address=0.0.0.0
    
    sudo service pdns-recursor start

<b>REFERENCES</b>

DNS Diag - https://dnsdiag.org</br>
DNS Wikipedia - https://en.wikipedia.org/wiki/Domain_Name_System</br>
PowerDNS - https://www.powerdns.com</br>
Unbound - https://www.nlnetlabs.nl/projects/unbound/about</br>
