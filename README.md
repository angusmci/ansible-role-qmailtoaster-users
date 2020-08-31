# qmailtoaster

This is an Ansible role for creating users and domains for a `qmailtoaster` install.

It complements the [`qmailtoaster`](https://github.com/angusmci/ansible-role-qmailtoaster) role, which implements a basic `qmailtoaster` setup.

Note that while the `qmailtoaster` role attempts to stay as close as possible to the install scripts developed by Eric Broch, this role represents a very opinionated approach to setting up users for a `qmailtoaster` system. This approach is powerful but can be complex.

For more information about `qmailtoaster`, see http://qmailtoaster.com.

THIS ROLE IS UNDER ACTIVE DEVELOPMENT AND HAS NOT BEEN EXTENSIVELY TESTED. PARTS MAY CHANGE WITHOUT WARNING. USE ENTIRELY AT YOUR OWN RISK. NO WARRANTY, EXPRESS OR IMPLIED.

## Requirements

TBD

## Configuration Variables

The behavior of the role is controlled by a complex variable structure that defines the domains and users to create.

Domain configuration allows you to specify the desired domains and users as a nested configuration structure. The structure specifies which domains to create, and which users to add to those domains.

The structure is passed to the role as a variable named `domains`. So an invocation of the role might look like:

	- name: set up qmailtoaster users
  	  include_role:
        name: ansible-role-qmailtoaster-users
      vars:
        domains: "{{ my_domains }}"

where `my_domains` is a variable (it’s generally easiest to define the structure in a separate `vars` file) that contains the full domain specification.

### Domains

A basic spec that only creates domains might look like:

	my_domains:
	  - name: example.com
	    password: "S3cr3t!"
	  - name: example.net
	    password: "S3cr3t!"
	    
This would add two domains (`example.com` and `example.net`) to the target server(s) and would create a postmaster account with the password "`S3cr3t!`", but wouldn't do anything else.

Domains are added using the `vadddomain` command. Because this command will throw an error if it is invoked on an already-existing domain, the task checks to see if a domain directory for the domain already exists in `/home/vpopmail/domains`. If it does, the task is skipped. If you need to recreate a domain, delete the domain directory (and clean up the `vpopmail` database by hand to avoid an error).

### Emails

The main purpose of the role is to set up email addresses on the servers. Each address will do one of four things:

delete
: Mail sent to this alias or group will be deleted unread.

deliver
: Mail sent to this alias or group will be delivered to a local user.

forward
: Mail sent to this alias or group will be forwarded to a remote address.

write
: Mail sent to this alias or group will be written to a specific Maildir directory.

These four choices are referred to as **actions**. In most cases, you’ll only need the first three. The fourth option, `write`, is included for the rare cases where you need to deliver mail to a location that isn’t the user’s top-level `Maildir`.

Mail handling in `qmail` is controlled by `.qmail` files. The `qmailtoaster-users` role’s main job is to create appropriate `.qmail` files for each address.

#### Default (‘catch-all’) account

Mail that is not addressed to a known user is handled as specified in the `.qmail-default` file. The most common way to handle this is by deleting it. A typical configuration will therefore look like this:

	my_domains:
	  - name: example.com
	    password: "S3cr3t!"
	    default:
	      action: delete
	      
The `action` property of `default` specifies how to handle mail: in this case, by simply deleting it.

Because deletion doesn't require any other information, that's all there is to setting up this catch-all.

There are other options as well. These typically require additional information. For example, you might want to forward messages sent to the catch-all to some other address.

	default:
	  action: forward
	  address: postmaster@hotmail.com
	  
or deliver it to a user at a domain on the local server:

	default:
	  action: deliver
	  user: fred
	  domain: example.net
	  
or write it to a specific location:

	default:
	  action: write
	  directory: /home/vpopmail/domains/example.com/fred/Maildir/.Junk/
	  
Note that each of these action types requires additional information. In the case of `forward`, you need to provide a forwarding address; in the case of `write` you need to provide a full path to a `Maildir` directory or subdirectory; in the case of `deliver` you need to provide a username (`user`) and a domain (`domain`) to identify where the message should be delivered. The domain must be local to the server; to deliver to an address at a different server, use `forward` instead.

#### Role accounts

After setting up a catch-all, the next step will probably be to define role accounts for the domain. Role accounts are standard addresses such as `webmaster` and `sales` that exist at most domains. These well-known accounts are defined in [RFC 2142](http://www.faqs.org/rfcs/rfc2142.html).

A typical domain might have anything from 6 (a minimal set) to 24 (a more complete list) role accounts. You could configure them all individually, but that’s a lot of work and makes for many lines in your configuration file. So this role groups the different accounts into **rolesets** as follows:

business
: info, marketing, sales, support

network
: abuse, noc, security

owner
: admin, billing

services
: hostmaster, mail, postmaster

webmaster
: webmaster
	
The idea is that at small organizations, each of the addresses in each group are likely to be handled by one person or department. For example, `info`, `marketing`, `sales` and `support` might all be handled by the same person or team, while `abuse`, `noc` and `security` might be the responsibility of a different person or team. Grouping the addresses like this simplifies your configuration files by allowing you to write handling directives for a group of addresses together, rather than one by one.

If this doesn’t work for your organization, you can still use the **aliases** feature to write rules for individual addresses. Alias definitions will override the definitions given for rolesets. 

An example definition might look like:

	my_domains:
	  - name: example.com
	    rolesets:
	      - name: business
	        action: forward
	        address: commercial@example.net
	      - name: network
	        action: forward
	        address: tech@example.net
	      - name: webmaster
	        action: forward
	        address: tech@example.net
	        
This would create `.qmail` files that forward any mail addressed to `info`, `marketing`, `sales` or `support` to `commercial@example.net`, while mail to `abuse`, `noc`, `security` and `webmaster` would be sent to `tech@example.net`. Mail sent to any other role account, such as `postmaster` would go to the catch-all account, because the definition above doesn't specify how the group that `postmaster` belongs to should be handled.

The `roleset_members` variable that defines all these groups also includes some other groups, including `other` (a long list of mostly specialized role accounts including `uucp` or `news`), `standard` (a standard set of the most commonly-used role accounts), `minimal` (just the accounts that most domains should support), and `all` (every account belonging to any of the groups).

If you’re a one-person shop running your own domain, you might want to have something like:

	my_domains:
	  - name: example.com
	    default:
	      action: delete
	    rolesets:
	      - name: minimal
	        action: deliver
	        user: me
	        domain: example.com
	        
which would deliver all mail sent to any of the standard role accounts to the mailbox `me` on `example.com`. Note that mail that isn’t to one of these accounts will be handled by the catch-all. So, for example, if you used the `minimal` set, then mail sent to `info@` would go to your mailbox, but mail sent to `feedback@` -- which is not in the `minimal` set -- would be deleted.

#### Users

The next part of the definition specifies actual user accounts for each domain. For the most part, these are users that have mailboxes on the local server. A partial definition including a user might look like:

	- name: example.com
	  users:
	  - name: fred
		comment: Fred Smith
		action: deliver
		password: "Fr3dz-p455w0rd"

Users are created using `vadduser`, which adds an entry to the `vpopmail` database and creates a `Maildir` directory for the user. Once again, because `vadduser` is not idempotent, the role uses the existence of the user’s `Maildir` directory as proof that the user has already been created and will not try to add the user a second time. One side-effect of this is that it means that you can’t change the user’s password by editing the definition and then re-running the playbook.

#### Aliases

The final part of the definition deals with **aliases**, which are email addresses that act as proxies for some other destination. Aliases don’t have local mailboxes of their own: they simply forward or deliver to some other ‘real’ user.

The definition of an alias follows the same format as other types of addresses. For example:

	- name: example.com
	  aliases:
	    - name: press
	      action: deliver
	      user: fred
	      domain: example.com
	    - name: fred.smith
	      action: deliver
	      user: fred
	      domain: example.com
	    - name: technology
	      action: forward
	      address: joe@example.net
	      
This definition creates three addresses: `press@example.com`, `fred.smith@example.com` and `technology@example.com`. The first two are aliases for the user `fred`, so that anything sent to `press` or `fred.smith` will be delivered to them; the third forwards to a remote user `joe` at `example.net` 

### Filtering

Definitions of users can also include the flag `filter`. If `filter` is set to `yes`, the role will create the infrastructure for user-specific `procmail` filters to filter the user’s mail before delivery or forwarding. Here is an example of a filtered configuration:

	- name: example.com
	  users:
	  - name: fred
	    comment: Fred Smith
	    action: deliver
	    filter: yes
	    password: "Fr3dz-p455w0rd"
	    filtersets:
		  - whitelist
		  - blacklist
		  - subscriptions
	    filterdirs:
		  - name: JUNKDIR
		    path: .Junk/
		  - name: BULKDIR
		    path: .Bulk/

In addition to creating a `Maildir` directory for the user `fred` in `/home/vpopmail/domains/example.com/fred`, this will also create a directory at `/home/vpopmail/domains/example.com/.procmail-fred` containing the files:

	rc.main
	rc.whitelist
	rc.blacklist
	rc.subscriptions
	
as well as two subdirectories `.Junk` and `.Bulk` in the user’s `Maildir` directory. The `rc.main` file will contain a skeletal `.procmail` setup something like:

	SHELL=/bin/sh

	PMDIR=/home/vpopmail/domains/example.com/.procmail-fred

	LOGFILE=/home/vpopmail/domains/example.com/.procmail-fred/log
	VERBOSE=off
	CRLF="
	"

	MAILDIR=/home/vpopmail/domains/example.com/fred/Maildir/
	JUNKDIR=/home/vpopmail/domains/example.com/fred/Maildir/.Junk/
	BULKDIR=/home/vpopmail/domains/example.com/fred/Maildir/.Bulk/

	INCLUDERC=$PMDIR/rc.whitelist
	INCLUDERC=$PMDIR/rc.blacklist
	INCLUDERC=$PMDIR/rc.subscriptions

	:0w
	$MAILDIR

This skeleton sets up variables that identify the path to the user’s main `Maildir` and to the `Junk` and `Bulk` subdirectories, and pulls in `procmail` recipes from the `rc.whitelist`, `rc.blacklist` and `rc.subscriptions` files. Those files are initially empty: you can add `procmail` recipes to them to filter mail and re-route it to the appropriate directory or subdirectory. Anything not filtered will be simply written to the user’s main `Maildir`.

If the flag `filter` is true, the `.qmail` file written for the user will look something like:

	| /var/qmail/bin/preline /bin/procmail -p -m /home/vpopmail/domains/example.com/.procmail-fred/rc.main
	
which routes incoming mail through `procmail`, using the user’s `rc.main` file.

Normally, filtered messages will be delivered to a local user (i.e. the `action` for the user will be `deliver`). However, you could also specify an action of `forward` and provide a remote email address as the value for `address`: in that case, incoming messages and any messages not filtered out by the provided filters will be forwarded to the remote address.

You can also set the `filter` flag when creating a catch-all account, rolesets or aliases. For example:

	my_domains:
	  - name: example.com
	    default:
	      action: deliver
	      filter: yes
	      user: fred
	      domain: example.com
	    rolesets:
	      - name: minimal
	        action: deliver
	        filter: yes
	        user: fred
	        domain: example.com
		aliases:
		  - name: fred.smith
		    action: deliver
		    filter: yes
		    user: fred
		    domain: example.com
		    
The one requirement is that the filtered action must `deliver` to a local user that was created with the `filter` flag set to true. Only the user-creation part of the role sets up the necessary filtering infrastructure. If you wrote, for example:

	my_domains:
	  - name: example.com
	    aliases:
	      - name: fred.smith
	        action: forward
	        filter: yes
	        address: fred.smith@example.net
	        
then the mail delivery system would go looking for a `procmail` recipe file that doesn't exist and delivery would fail.

## Example Playbook

Here is an example of invoking the role.

	tasks/main.yml
	==============
	
    - name: set up users
      include_role:
    	name: ansible-role-qmailtoaster-users
      vars:
    	domains: {{ my_domains }}
    
    vars/main.yml
    =============
    	
    my_domains:
      - name: example.com
        password: "S3cr3t!"
        default:
          action: delete
        rolesets:
          - name: minimal
            action: deliver
            user: fred
        users:
          - name: fred
            comment: "Fred Smith"
		    password: "Fr3dz-p455w0rd"  
            action: deliver
    		filter: yes  
    		filtersets:  
      		  - whitelist  
      		  - blacklist  
      		  - subscriptions  
    		filterdirs:  
      		  - name: JUNKDIR  
        		path: .Junk/  
      		  - name: BULKDIR  
        	    path: .Bulk/
    	aliases:
    	  - name: fred.smith
    	    action: deliver
    	    filter: yes
    	    user: fred
    	    domain: example.com
    	  - name: mary.brown
    	    action: forward
    	    address: mary@example.net
    	
## License

BSD

## Author Information

Ansible role

- Angus McIntyre -- http://nomadcode.com/
