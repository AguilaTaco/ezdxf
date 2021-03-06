New Setup Process
=================

The actual setup process of DXF structures for new documents or for loading 
DXF files is not well designed, because of the lack of understanding of this 
structures and dependencies between all the DXF types in the beginning of 
the development and the DXF reference is also not very helpful for this topic.

The idea is to create a new setup design:

1. There are two basic scenarios:
    - Load structures from file: `LOAD`
    - Create new structures by ezdxf: `CREATE`
1. The default constructor for entities do not get a reference to the actual 
    DXF document. The new entity is a virtual entity.
1. Binding a virtual entity to a DXF document is different for the 2 scenarios:
    - `LOAD`: all required resources should be loaded at the time of binding the
      entity, als handles are set and can be validated
    - `CREATE`: some resources should be present (e.g. linetypes, text-styles)
      and some required resources should be created (e.g. ImageDef Reactor, 
      SEQEND)

The new setup process should also consider the following loading scenarios:

- DWG loader add-on
- iterdxf add-on

Which means the `factory` module should provide the necessary functions to 
create/load entities for these add-ons.

## Terminology

### Virtual Entity

- not stored in the entity database of a document: DXF attribute `handle` is `None`
- not linked to a layout/owner: DXF attribute `owner` is `None`

### Unlinked Entity

- stored in an entity database = bound to a document
- not linked to a layout/owner: DXF attribute `owner` is `None`

```
    Virtual Entity == Unlinked Entity
    Unlinked Entity != Virtual Entity
```

## DXFEntity Design

Remove dependency of DXF entities from DXF document:

- Remove `doc` attribute 
- Remove `dxffactory` attribute 
- Remove `entitydb` attribute

## LOAD

The loading process has two stages:

### First Stage

Load content from file/stream and store them in a DXF structure database. 

### Second Stage

Parse DXF structure database:

- Create sections: HEADER, CLASSES, TABLES, BLOCKS and OBJECTS, the ENTITIES 
  section is a relict from older DXF versions and has to be exported including 
  the modelspace and active paperspace entities, but all entities reside in a 
  BLOCK definition, even modelspace and paperspace layouts are only BLOCK 
  definitions and ezdxf has no explicit ENTITIES section.
- Create layouts: Blocks, Layouts
    - Bind entities to the document: `factory.bind_loaded()`
    - Link entities to a layout: `Layout.link()`

## CREATE

A new entity is always a unbounded and virtual entity after instantiation:

- DXF owner is `None`
- DXF handle is `None`

## BIND

Binding the entity to a document does:

- store entity in the document entity database and set DXF attribute `handle` 
- DXF attribute `owner` is still `None`, ia not linked to any layout

Without an assigned layout or parent entity the entity is an unlinked entity, 
but bound to a document, this means it is possible to check or create required 
resources.

## LINK

This makes an entity to a real DXF entity, which will be exported 
at the saving process. Any DXF entity can only be linked to **one** parent
entity like DICTIONARY or BLOCK_RECORD.

DXF Objects:

- link to OBJECTS section by adding entity to a parent entity in the OBJECTS 
  section, most likely a DICTIONARY object and store entity in the entity 
  space of the OBJECTS section
- Extension dictionaries can also own entities in the OBJECTS section  

DXF Entities:

- link entity to a layout by `BlockRecord.link(entity)`, which set the `owner`
  handle to BLOCK_RECORD handle (= layout key) and store entity in entity space 
  of the BLOCK_RECORD
- set paperspace flag

# Factory module

Decouple `EntityFactory()` from a specific document, this is a relict from the 
older ezdxf design until v0.9, where each DXF version had its own factory class.
In fact the `EntityFactory()` object is obsolete, `next_underlay_key()` should 
be moved to ..., `next_image_key()` is not used. The property `dxffactory` 
should be removed from all objects.

## Factory functions

- `new(dxftype, dxfattribs)`, create a new virtual DXF object/entity
- `load(tags)`, load (create) virtual DXF object/entity from DXF tags
- `bind(entity, doc)`, bind an entity to a document, create required 
  resources if necessary (e.g. ImageDefReactor, SEQEND) and raise exceptions for
  non-existing resources.
  For adding loaded or foreign entities see below, for entities created by a 
  package-user raise an exception to informed about the invalid package usage.
- bind loaded and foreign entities: 
  1. bind entity loaded from a file to a document, all referenced resources must 
     exist, but try to repair as many flaws as possible, because this issues 
     were created by another application and are not the responsibility of the 
     package-user.
  2. bind an entity from another document, all invalid resources will be 
     removed silently or created (e.g. SEQEND). This is a simple import from 
     another document without resource import for a more advanced import 
     including resources exist the `importer` add-on.
     
  Create an `Auditor()` and repair the entity, if unrecoverable errors exist:
  log the problem and kill the entity. Log applied fixes.
  This requires an fully initialized and valid DXF document.
- Bootstrap problem for binding loaded table entries and objects in the OBJECTS 
  section! Can't use `Auditor()` to repair this objects, because the DXF 
  document is not fully initialized.
- `is_bound(entity, doc)` returns True if `entity` is bound to document `doc`
- `cls(dxftype)`, returns the class
- `register_entity()`, registration decorator  
- `replace_entity()`, registration decorator

## Class Interfaces

### Entities

1. `CREATE` interface as class method
1. `LOAD` interface as class method
1. `DESTROY` interface to kill an entity, set entity `STATE` to "dead", which 
   means `entity.is_alive` returns False. All entity iterators like 
   `EntitySpace`, `EntityQuery`,  and `EntityDB` must filter (ignore) "dead" 
   entities. Calling `DXFEntity.destroy()` is the normal way to delete entities.

```Python
from ezdxf.entities.dxfentity import DXFNamespace

class DXFEntity:
    def __init__(self):
        self.dxf = DXFNamespace()

    @classmethod
    def new(cls, dxfattribs):
        """ CREATE interface """

    @classmethod
    def load(cls, tags):
        """ LOAD interface """

    @property
    def is_alive(self):
        """ STATE interface """
        return hasattr(self, "dxf")

    @property
    def is_virtual(self):
        """ STATE interface """
        return self.dxf.handle is None

    @property
    def is_bound(self):
        """ STATE interface """
        return self.dxf.handle is not None

    @property
    def is_linked(self):
        """ STATE interface """
        return self.dxf.owner is not None

    def destroy(self):
        """ DESTROY interface """
```

### Layouts

1. `LINK` interface to assign a layout to an entity 
1. `UNLINK` interface to remove a layout assignment from an entity
1. Layouts have back-link `doc` to the DXF document
1. Support for a virtual layout, which can store virtual entities
1. It is not possible to move or copy layouts between documents, 
   maybe use `importer` add-on
1. `copy_to_layout(entity, layout)` module function to copy an entity to another layout 
1. `move_to_layout(entity, layout)` module function to move an entity to another layout 

```Python
from ezdxf.entities import factory
from ezdxf import audit

class Layout:
    doc: 'Drawing' = None  # back-link to DXF document

    def add_entity(self, entity):
        """ LINK interface """
    # cleanup interface:
    link = add_entity

    def unlink(self, entity):
        """ UNLINK interface """

    def add_foreign_entity(self, entity):
        """ LINK foreign entity """
        auditor = audit.audit(entity, self.doc)
        if not auditor.has_errors:
            factory.bind(entity, self.doc)
            self.add_entity(entity)
    # cleanup interface:
    link_foreign = add_foreign_entity
    
    def purge(self):
        """ Remove dead entities. """

    def move_to_layout(self, entity, layout):
        """ deprecated """
        # replacement: module function
        move_to_layout(entity, layout)

```

### Database

1. `BIND` interface to bind entity to the database and document
1. `UNBIND` interface to remove an entity from the database and set the entity 
   state to a virtual entity, which should also `UNLINK` the process, because an 
   layout can not store a virtual entity.
1. remove/deprecate `delete_entity()` interface, which is the same as `UNBIND` 
   and `DESTROY` entity

```Python
class EntityDB:
    def add(self, entity):
        """ BIND interface """
    # cleanup interface:
    bind = add

    def unbind(self, entity):
        """ UNBIND interface """

    def delete_entity(self, entity):
        """ deprecated """
        entity.destroy()  # replacement

    def purge(self):
        """ Remove dead entities. """

```