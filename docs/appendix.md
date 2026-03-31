# Appendix

This appendix provides an overview of the process of migrating from the old Security Onion 2.4 to the new Security Onion 3.

!!! TIP
    
    If you are a current Security Onion Solutions customer with Professional Services or Appliance coverage, contact SOS support and we can help you through this process.

!!! WARNING
    
    Security Onion 3 only supports Oracle Linux 9. If you are running Security Onion 2.4 on some other unsupported distro, then you will need to perform a fresh installation of Security Onion 3.

!!! WARNING
    
    We recommend trying this process in a test environment before attempting in your production environment.
    
If you have reviewed all of the warnings above and still want to attempt migration, you should be able to do the following.

!!! NOTE
    
    If you have a distributed deployment, you will need to perform the steps on the manager first and then on each of the remaining nodes.

First, make sure that your 2.4 installation is fully updated via [soup](soup.md):

```
sudo soup
```

Next, make sure there is a backup in /nsm/backup:

```
sudo ls -alh /nsm/backup
```

Once you have confirmed your backup, you may update to Security Onion 3. 

To begin, you will need to update to the new version of soup:

```
sudo soupto3
```

Once you have the new version of soup, then you will need to run it to perform the upgrade:

```
sudo soup
```
