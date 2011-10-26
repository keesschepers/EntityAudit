# EntityAudit Extension for Doctrine2

This extension for Doctrine 2 is inspired by [Hibernate Envers](http://www.jboss.org/envers) and
allows full versioning of entities and their associations.

This fork is specially forked for support in Zend Framework.

## How does it work?

There are a bunch of different approaches to auditing or versioning of database tables. This extension
creates a mirroring table for each audited entitys table that is suffixed with "_audit". Besides all the columns
of the audited entity there are two additional fields:

* rev - Contains the global revision number generated from a "revisions" table.
* revtype - Contains one of 'INS', 'UPD' or 'DEL' as an information to which type of database operation caused this revision log entry.

The global revision table contains an id, timestamp, username and change comment field.

With this approach it is possible to version an application with its changes to associations at the particular
points in time. 

This extension hooks into the SchemaTool generation process so that it will automatically
create the necessary DDL statements for your audited entities.

## Installation (In Zend Framework Application)

1. Intergrate Doctrine
You should intergrate Doctrine (if you didn't yet) in your Zend Framework application. A good suggestion is the Bisna 'glue' which you can check out here:
https://github.com/guilhermeblanco/ZendFramework1-Doctrine2

2. Add the library
Create a directory library/SimpleThings/EntityAudit and add this library to it.

3. Configure your application.ini and add:

	autoloaderNamespaces[] = "SimpleThings"

	pluginPaths.SimpleThings\EntityAudit\Application\Resource\ = "SimpleThings/EntityAudit/Application/Resource"

	;Add your entity paths here you want to audit
	
	resources.audit.auditedEntityClasses[] = "My\Entity\SomeEntity" 
	
	resources.audit.auditedEntityClasses[] = "My\Entity\SomeOtherEntity"
	
	;If you like you can apply these settings to:
	
	resources.audit.revisionTableName = "revisions"
	
	resources.audit.revisionTypeFieldName = "revtype"
	
	resources.audit.revisionFieldName = "rev"
	
4. Finally create the schema tables with the Doctrine commandline tool. If you are using bisna:

`php /path/to/your/zf/app/scripts/doctrine.php orm:schema-tool:update --dump-sql` to see the new tables in the update schema queue.

`php /path/to/your/zf/app/scripts/doctrine.php orm:schema-tool:update --force` to really create them

Notice: EntityAudit currently only works with a DBAL Connection and EntityManager named "default".

## Usage 

Querying the auditing information is done using a `SimpleThings\EntityAudit\AuditReader` instance.

In Symfony2 the AuditReader is registered as the service "simplethings_entityaudit.reader":

	<?php

	class IndexController extends Zend_Controller_Action
	{
		public function indexAction()
	        {	
			$entityManager = $this->getInvokeArg('bootstrap')->getResource('doctrine')->getEntityManager();
			$auditManager = $this->getInvokeArg('bootstrap')->getResource('Audit');
			$auditReader = $auditManager->createAuditReader($entityManager);
		}
	}

	?>

### Find entity state at a particular revision

This command also returns the state of the entity at the given revision, even if the last change
to that entity was made in a revision before the given one:

    <?php
    $articleAudit = $auditReader->find('My\Entity\SomeEntity', $id = 1, $rev = 10);

Instances created through `AuditReader#find()` are *NOT* injected into the EntityManagers UnitOfWork,
they need to be merged into the EntityManager if it should be reattached to the persistence context
in that old version.

### Find Revision History of an audited entity

    <?php
    $revisions = $auditReader->findRevisions('My\Entity\SomeEntity', $id = 1);

A revision has the following API:

    class Revision
    {
        public function getRev();
        public function getTimestamp();
        public function getUsername();
    }

### Find Changed Entities at a specific revision

    <?php
    $changedEntities = $auditReader->findEntitesChangedAtRevision( 10 );

A changed entity has the API:

    <?php
    class ChangedEntity
    {
        public function getClassName();
        public function getId();
        public function getRevisionType();
        public function getEntity();
    }

## Setting the Current Username

If your Zend_Auth identity is a string (Zend_Auth_Adapter_DbTable) or implements a __toString function this will be considered as the current user else you c$

        <?php
        protected function _initInjectUsernameToAuditConfiguration()
        {
                $this->bootstrap('audit');

                $auditConfiguration = $this->getResource('audit');
                $auditConfiguration->setCurrentUsername = 'some-user-name-here';
        }
        ?>

## TODOS

* Currently only works with auto-increment databases
* Proper metadata mapping is necessary, allow to disable versioning for fields and associations.
* It does NOT work with Joined-Table-Inheritance (Single Table Inheritance should work, but not tested)
* Many-To-Many assocations are NOT versioned
