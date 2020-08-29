# qmailtoaster

This is an Ansible role for creating users and domains for a `qmailtoaster` install.

It complements the [`qmailtoaster`](https://github.com/angusmci/ansible-role-qmailtoaster) role, which implements a basic `qmailtoaster` setup.

Note that while the `qmailtoaster` role attempts to stay as close as possible to the install scripts developed by Eric Broch, this role represents a very opinionated approach to setting up users for a `qmailtoaster` system. This approach is powerful but can be complex.

For more information about `qmailtoaster`, see http://qmailtoaster.com.

BECAUSE THIS ROLE IS STILL UNDER ACTIVE DEVELOPMENT, IT IS NOT RECOMMENDED FOR PRODUCTION SYSTEMS. NO WARRANTY, EXPRESS OR IMPLIED.

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

### Emails

The main purpose of the role is to set up email addresses on the servers. Each address will do one of four things:

delete
: Mail sent to this alias or group will be deleted unread.

deliver
: Mail sent to this alias or group will be delivered to a local mailbox.

forward
: Mail sent to this alias or group will be forwarded to a remote address.

filter
: Mail sent to this alias or group will be filtered using `procmail` and then delivered.

Mail handling in `qmail` is controlled by `.qmail` files, so the main function of the role is to create a `.qmail` file for each address.

#### Default (‘catch-all’) account

Mail that is not addressed to a known user is handled as specified in the `.qmail-default` file. The most common way to handle this is by deleting it. The typical configuration looks like this:

	my_domains:
	  - name: example.com
	    password: "S3cr3t!"
	    default:
	      action: delete
	      
The `action` property of `default` specifies how to handle mail: in this case, by simply deleting it.

Because deletion doesn't require any other information, that's all there is to setting up this catch-all.

There are other options. For example, you might want to forward messages sent to the catch-all to some other address.

	default:
	  action: forward
	  address: postmaster@hotmail.com
	  
or deliver it to a user at a domain on the local server:

	default:
	  action: deliver
	  user: fred
	  domain: example.net
	  
or filter it before delivery:

	default:
	  action: filter
	  user: fred
	  domain: example.net
	  
Filtering uses `procmail`: the role can also help with setting up `procmail` filtering for users.

Note that each of these action types requires additional information: in the case of `forward`, you need to provide the forwarding address, while in the case of `deliver` or `filter` you need to provide a username (`user`) and a domain (`domain`) to identify where the message should be delivered. The domain must be local to the server; to deliver to an address at a different server, use `forward` instead.

#### Role accounts

After setting up a catch-all, the next step will probably be to define role accounts for the domain. Role accounts are standard addresses such as `webmaster` and `sales` that exist at most domains. These well-known accounts are defined in [RFC 2142](http://www.faqs.org/rfcs/rfc2142.html).

A typical domain might have anything from 6 (a minimal set) to 22 (a more complete list) role accounts. You could configure them all individually, but that’s a lot of work and makes for many lines in your configuration file. So this role groups the different accounts into sets as follows:

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
	
The idea is that, for example, the `business` set groups together four role accounts that -- at least at a small organization -- might be handled by the same person or department, the `network` set groups together technical accounts such as `abuse` and `security`, and so on. You can then write a definition for the group, rather than having to write one for each individual account. For example:

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
	        
would create `.qmail` files that forward any mail addressed to `info`, `marketing`, `sales` or `support` to `commercial@example.net`, while mail to `abuse`, `noc`, `security` and `webmaster` would be sent to `tech@example.net`. Mail sent to any other role account, such as `postmaster` would go to the catch-all account, because the definition above doesn't specify how the group that `postmaster` belongs to should be handled.

The `roleset_members` variable that defines all these groups also includes some other groups, including `other` (specialized role accounts including `uucp` or `news`), `standard` (a standard set of the most commonly-used role accounts), `minimal` (just the accounts that most domains should support), and `all` (every account belonging to any of the groups).

If you’re a one-person shop running your own domain, you might want to just have something like:

	my_domains:
	  - name: example.com
	    rolesets:
	      - name: standard
	        action: deliver
	        user: me
	        domain: example.com
	        
which would deliver all mail sent to any of the standard role accounts to the mailbox `me` on `example.com`.
	 
-- TO CONTINUE --


## Example Playbook

Here is an example of invoking the role.

    - name: set up users
      include_role:
    	name: ansible-role-qmailtoaster-users
      vars:
    	qmt_domains:
    	  - name: example.com
    		password: "{{ my_password6 }}"
    		default:
    		  action: delete
    		business:
    		  action: filter
    		  user: fred
    		  host: example.com
    		owner:
    		  action: filter
    		  user: bob
    		  host: example.com
    		network:
    		  action: filter
    		  user: jane
    		  host: example.com
    		services:
    		  action: filter
    		  user: elizabeth
    		  host: example.com
    		webmaster:
    		  action: filter
    		  user: fred
    		  host: example.com
    	  - name: example.net
    		password: "{{ my_password6 }}"
    		default:
    		  action: delete
    		business:
    		  action: none
    		owner:
    		  action: filter
    		  user: bob
    		  host: example.com
    		network:
    		  action: filter
    		  user: jane
    		  host: example.com
    		services:
    		  action: filter
    		  user: elizabeth
    		  host: example.com
    		webmaster:
    		  action: none

## License

BSD

## Author Information

Ansible role

- Angus McIntyre -- http://nomadcode.com/
