# Appendix

This appendix provides an overview of the process of upgrading from the old Security Onion 2.4 to the new Security Onion 3.

!!! TIP
    
    If you are a current Security Onion Solutions customer with Professional Services or Appliance coverage, contact SOS support and we can help you through this process.

!!! WARNING
    
    Security Onion 3 only supports Oracle Linux 9. If you are running Security Onion 2.4 on some other unsupported distro, then you will need to perform a fresh installation of Security Onion 3.

!!! WARNING
    
    We recommend trying this process in a test environment before attempting in your production environment.
    

## Check Backup

If you have reviewed all of the warnings above, next make sure there is a backup in /nsm/backup:

```
sudo ls -alh /nsm/backup
```

Once you have checked your backup and are ready to upgrade, you should be able to do the following. There are separate sections for Internet deployments and Airgap deployments.

## Internet Deployments

If your deployment has Internet access, first make sure that it is fully updated to version 2.4.211 via [soup](soup.md):

```
sudo soup
```

Once your deployment has been upgraded to 2.4.211, then run the following command:

```
sudo soupto3
```

This command will check some settings on your system and then download a new version of soup. You can then run the new version of soup to perform the upgrade:

```
sudo soup
```

## Airgap Deployments

If your deployment does not have Internet access and is in airgap mode, first make sure that it is fully updated to version 2.4.211 using [soup](soup.md#airgap) and the 2.4.211 ISO image.

Once that is done, then you will need to manually do a few things to prepare for the upgrade:

- make sure that the ``pcapengine`` setting is set to ``SURICATA``
- delete any old stenographer data

Finally, you can upgrade to version 3.0 using [soup](soup.md#airgap) and the Security Onion 3 ISO image. 
