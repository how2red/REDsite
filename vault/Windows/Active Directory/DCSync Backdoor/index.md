## SpectreOps' DCSync Backdoor

A **DCSync backdoor** exploits how Active Directory handles replication. By obtaining two specific permissions on the domain root (_DS-Replication-GetChanges_ and _DS-Replication-Get-Changes-All_), an account can request the same data that domain controllers share during replication. This includes password hashes for any account in the domain. Many believe full Domain or Enterprise Admin rights are required, but in reality, only those two ACEs on the domain object are necessary.

In this walkthrough, a normal computer account (`H2R-Box-3$`) is converted into a long-term **patsy**. This machine object is granted replication rights that allow it to access KRBTGT-related data, then hidden within the directory using ACE manipulation techniques described by SpecterOps in _An Ace Up The Sleeve_. Control of the object is delegated to an unprivileged user, reducing the likelihood of detection. Once established, the patsy serves as a persistent replication pivot that blends in with normal AD activity and remains unnoticed unless domain ACLs are carefully reviewed.


## Grant replication rights

1. Open **Active Directory Users and Computers**.
    
2. Navigate to the domain root, for example `how.2.red`.
    
3. Right-click the domain root and choose **Properties**.
    
    - Object only
        
    
> [!Tip]  
> You will need to add **Computers** to the Object Types list.

4. Select **Check Names**.
    
5. Click **Advanced** → **Add** → **Select a Principal**. Enter `H2R-Box-3$`, set **Type** to _Allow_, and set **Applies to** to _This object_.
    
6. In the permissions list, locate and select **Replicating Directory Changes** and **Replicating Directory Changes All**.
    
7. Click **Apply**.
    

---

## Now hide your patsy machine

1. Create a new OU: right-click the domain root `how.2.red` → **New** → **Organizational Unit**. Any name will work; in this example it is **Schema Extensions**.
    
2. When naming the OU, uncheck **Protect container from accidental deletion**.
    
3. In the OU’s **Properties**, enable **ShowInAdvancedViewOnly**.
    
4. Move `H2R-Box-3$` into the newly created OU.
    
5. Right-click `H2R-Box-3$` → **Security** → **Advanced**.
    
> [!Danger]  
> These next steps must be performed together or you risk locking yourself out.
    
6. Change the owner of `H2R-Box-3$` to a user account you control. Preferably use an unprivileged account you created or otherwise control and for which you have credentials.
    
7. Add the **Everyone** group and set an explicit **Deny** for **Full Control**. Make sure this applies to _This object and all descendant objects_.
    
8. Click **Apply**.
    
9. Repeat the same owner change and Everyone deny on the OU: change the owner to the user account you control, add a **Deny** for **Everyone**, check **List contents**, and ensure it applies to _This object and all descendant objects_.
    

---

When hidden this way, the machine account will not appear during LDAP enumeration for principals other than the patsy. It will reside in an OU that only the patsy can read.