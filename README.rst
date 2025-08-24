node-ci
#######

Description
===========

A Dockerized build of `NCI <https://github.com/node-ci/nci>`_ designed to **bootstrap its own secrets** and run without manual GPG setup.

There's a project called **000-secrets** written to get the passwords and make them available to all the projects.

On the **first run**, the project:

- Creates a GPG keypair (dedicated for this instance).
- Initializes or clones a password-store repository.

Now we need to tweak our passwords so they can be decrypted by the new key

- import the key
- trust the key
- rencrypt the repo
- git push to remote


Back to the container, on the **second run**, the project:

- Reuses the same keypair.
- Decrypts and uses secrets from the password-store.

This makes it easy to spin up NCI instances with **independent identities** and **granular password-stores**.

---

Features
--------

- 🚀 **Turnkey setup**: no need to import your own GPG keys.
- 🔑 **Granularity by design**: each container can use a different password-store repository.
- 🔄 **Recovery-friendly**: if you delete volumes, a new keypair + store will be initialized.
- 🔧 **Flexible**: override repo URL and password-store directory via environment variables.
- 🐳 **Docker integration**: build and run containers as part of CI pipelines.


1) Installation - Run as it is (demo)
=====================================

.. code-block::

    git clone https://github.com/nunogrl/node-ci
    cd node-ci
    docker-compose up -d


Go to `localhost:3007 <http://localhost:3007>`_.

This starts node-ci using a public password repository (for demonstration).

To use your own secrets, continue with the steps below.


How the keys work (at a glance)
-------------------------------

- Public keys[🔑]  **encrypt**.
- Private Keys [🔐] **decrypt**.


.. mermaid::

    flowchart RL
      subgraph Repo[github private repo]
        C[(git repo)]
      end
      subgraph NodeCI[Node-CI server]
        NCpriv[Node-CI private 🔐]
      end
      subgraph User
        Upriv[User
             private 🔐]
        
        NCpub[Node-CI
            public 🔑
        imported]
    
        Upub[User
            public 🔑]
      end
    
      Upub -- used to encrypt --> C
      NCpub -- used to encrypt --> C
      C -- decrypts with --> Upriv
      C -- decrypts with --> NCpriv


You (locally) add **Node-CI’s public key** to your password store and **reencrypt**.

Node-CI (server) keeps **only its private key** to decrypt at build time.


Pre-flight — create your own password store (locally)
-----------------------------------------------------


Requires ``gpg`` and ``pass`` on your workstation.

.. code-block:: console

    # Adjust if you want a non-default location (default is ~/.password-store)
    export PASSWORD_STORE_DIR="${VAULT_DIRECTORY}"
    
    # Output secrets in ASCII-armored text (safer for piping/logs)
    export PASSWORD_STORE_GPG_OPTS=--armor
    
    # Initialize your password store with your GPG identity
    pass init you@domain.com
    
    # Add a few entries
    pass generate --no-symbols -f server1/site-test/alpha 16
    
    # Put it under version control
    pass git init
    # Create your (private) remote and connect it
    pass git remote add origin git@github.com:myorg/order66-secrets
    pass git push --set-upstream origin master
    
    # Add more secrets whenever you like
    pass generate --no-symbols -f server2/site-test/beta 16
    pass generate --no-symbols -f server-beta/ssh/root 16
    pass git push


2) Point node-ci to your repo
=============================

Start fresh (or edit your cloned repo) and set the project to your values:

- docker-compose.yaml

  - ``SECRET_REPO=...`` → your password-store Git URL

  - ``CICD_ID=Toaster`` → key “name” to be generated for node-ci

  - ``CICD_MAIL=toaster@nunogrl.com`` → email for that key

- 000-secrets/config.yaml

  - ``repository: ...`` → same repo URL as above

Auth to your repo:

  - Put the SSH private key (or token) in the ``ssh/`` folder (and add ``ssh_config`` and ``known_hosts`` as needed).


3) Bring it up and export Node-CI’s public key
==============================================


.. code-block:: console

    docker-compose up -d


In the web UI ( `<http://localhost:3007>`_ ):

  - **Run project** ``000-secrets`` (first run generates a GPG keypair server-side and prints the public key).
  - Copy that **public key** into node-ci.asc on your workstation, then:


.. code-block:: console

    gpg --import node-ci.asc
    gpg --edit-key "toaster@nunogrl.com"
    # inside the gpg prompt:

    gpg> trust
    (...)
      1 = I don't know or won't say
      2 = I do NOT trust
      3 = I trust marginally
      4 = I trust fully
      5 = I trust ultimately
      m = back to the main menu
    
    Your decision? 5
    Do you really want to set this key to ultimate trust? (y/N) y
    
    pub  rsa4096/2FC332B787563BEB
         created: 2025-08-24  expires: never       usage: SCEAR
         trust: ultimate      validity: unknown
    sub  rsa4096/F5A866722097FDBC
         created: 2025-08-24  expires: never       usage: SEA 
    [ unknown] (1). Toaster (Server Usage) <toaster@nunogrl.com>
    Please note that the shown key validity is not necessarily correct
    unless you restart the program.
    
    gpg> quit



Why “ultimate”?
---------------


This key is used purely to **decrypt** inside a server.

If that server is ever compromised, you’ll **rotate** the key and **reencrypt the store**.

*Trust* here simply instructs your local GPG that it’s OK to encrypt to this key without interactive prompts.


4) Re-encrypt your password store to include Node-CI
====================================================

Reinitialize recipients to include your usual users **plus** the Node-CI key:


.. code-block:: console

    # Rewrites .gpg-id and reencrypts all entries to the listed recipients
    pass init user1 user2 user3 user4 "toaster@nunogrl.com"
    pass git push


    **Tip:** The current recipients are listed in ``~/.password-store/.gpg-id``.



5) Run 000-secrets again and verify decryption
==============================================

Trigger 000-secrets a second time. You should see a successful decryption check at the end of the logs (no secrets are printed):

.. code-block:: console

    Testing decryption of a password:
    stderr: gpg: encrypted with rsa4096 key, ID F5A866722097FDBC, created 2025-08-24
          "Toaster (Server Usage) <toaster@nunogrl.com>"
    stderr: gpg: encrypted with ELG key, ID 87111A0115766816


From now on, pipelines can safely call ``pass path/to/entry`` during builds.


Notes & tips
============

If you **wipe** the server volumes, Node-CI will **generate a new keypair** on
the next run; just repeat **steps 3–4** to reauthorize it.

If you truly want multiple node-ci instances to share the same identity, you
can copy the ``.gnupg`` directory between them — but the **recommended** approach is
**separate identities** per container/store.

For private Git hosts, remember to add host keys to ``ssh/known_hosts`` or use
``StrictHostKeyChecking=no`` in a controlled environment.
