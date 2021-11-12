
Role Variables
--------------
    list_database:
      - BTC
      - BNB
      - SHIB

    list_database_remove:
      - DOGE

    list_user:
      - 
        username: CZ
        password: 1
        role_attr_flags: SUPERUSER
        
      - 
        username: ELONMUSK
        password: 1
        role_attr_flags: CREATEDB
      - 
        username: BINACE
        password: 1
        role_attr_flags: LOGIN

    list_user_deleted:
      - SLP
      - C98
    
    list_privilege:
      -
        role_name: BTC
        db_name: BTC
        objs_type: ALL_IN_SCHEMA
        privs_list: SELECT,INSERT,UPDATE

      -
        role_name: BTC
        db_name: BNB
        objs_type: ALL_IN_SCHEMA
        privs_list: SELECT

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - {role: psql-database, tags: psql-database}

License
-------

BSD
