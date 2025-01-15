####################################
Column groups in the SDM and the RSP
####################################

.. abstract::

   This note describes proposed uses for column groups in the Science Data Model and how these might be used in the Rubin Science Platform.

Motivation
==========

We have long planned to use the IVOA VOTable "column group" feature (VOTable XML ``<GROUP>``\) in
the Rubin "Science Data Model" and in the behavior of the Rubin Science Platform (RSP).

"Column groups" are non-hierarchical, it's important to recognize, so any given column can
belong to several groups, or none.
This has advantages for the planned use cases, but also introduces complications in UI
design for making users aware of the presence of groups.

The motivation for the use of groups in the SDM and RSP has been two-fold:

* Annotate small sets of columns that form a composite entity, essentially an object,
  that could be treated as a unit by a client application (like Firefly or TOPCAT),
  or a client library (like populating an object model in PyVO/Astropy).
  The reference use cases for this have always been: a) to associate a column, or a set
  of columns, with its uncertainties, or error matrix; and b) to designate a set of
  columns as a parallax plus proper motion model.
  The aim is to allow client software to recognize these entities in a principled and
  interoperable way without relying on column-naming conventions that differ from
  project to project.
* Annotate larger and less-structured groups of columns as related, e.g., as all
  arising from a particular algorithm in the science pipelines -- exemplified by
  the different categories of measurement algorithms in the ``Object`` table
  (e.g., ``cModel`` fluxes).

Exploitation of this mechanism requires work in several areas:

* Annotation of the SDM, expressed in Felis, with groups, including group-level
  metadata providing a semantic identity for the groups;
* Transfer of this information from Felis to a form usable by RSP services, most
  notably TAP;
* Implementation of the annotation of VOTable results, from TAP but potentially
  also from other RSP services (like forced photometry on demand, or light curve
  services), with the group data, following the VOTable standard;
* Exploitation of groups in the RSP Portal (Firefly); and
* Exploitation of groups in PyVO/Astropy.

Relationship with VO-DML and MIVOT
----------------------------------

The IVOA ``<GROUP>`` mechanism is in some respects both a historical precursor and
functional subset of the capabilities of the more recently defined VO-DML data model
language and the MIVOT model for serialization of VO-DML in VOTable.

Realistically we are not going to be able to adopt VO-DML and MIVOT into the SDM,
or meaningfully support them in Firefly, before at least 2026, if then, and their
use will require a well-resourced effort across multiple groups.
It's not clear -- further thought is needed -- whether this would even be compatible
with continuing to use Felis for the SDM.
It's possible that we might have to replace Felis with parts of the technology
stack around VO-DML, and that would be quite a disruptive change.

Additionally, it's unlikely that IPAC/IRSA would adopt VO-DML and MIVOT on any
near-term time scale, whereas IRSA's TAP service already supports ``<GROUP>``\s
and there is some very basic support for them in Firefly already.
The ``<GROUP>`` approach therefore is more likely to benefit from the effort
multiplier of collaboration with IRSA on Firefly development.

The ``<GROUP>`` mechanism will allow us to meet a number of key use cases before
that time, and I believe will also help us understand whether the substantially
larger effort involved in VO-DML and MIVOT would be justified by the benefits to
science users.

The VOTable <GROUP> Model
=========================

We describe briefly the nature of the ``<GROUP>`` model in the VOTable standard.

Note that because of recent (2024) work on standardization, we now have a way
to include this kind of rich metadata in Parquet files, not only traditional
VOTable files, thus enabling exposing this information in the next-to-the-data
processing environment.

Defining group membership
-------------------------

Groups are defined by ``<GROUP>`` elements in the XML data model, which generally
appear as peers to the ``<FIELD>`` elements.
Groups may be composed of a combination of fields and ``<PARAM>``\s.
In this version of this note we consider only fields, though there are
interesting applications for ``<PARAM>``\s and a future revision should try to
address them as well.

Fields (commonly, table columns) are included in groups by reference, not
hierarchically, so a field may appear in many groups, or one, or none.
Groups may be contain other groups, but this is "by value" and hierarchical;
there is no such thing as a reference to a group, and a group cannot be
defined as belonging to more than one other group.

Groups are a property of a table's columnar data model as a whole;
the same group structure applies to each row in a table, and *rows* cannot
be grouped.

Group-level metadata
--------------------

Groups may have a name, a description, a UCD, and a UType.
It was originally intended for the UType to be usable to express a reference
to an externally known data model for the group, which could be as simple as
"a value with an uncertainty" or could express complex object-oriented data.

In practice there are some IVOA-standard data models that are expressible
with well-defined interoperable UTypes, but they don't cover the space interesting
to us very well.
We are very likely to need to define our own UTypes.

In some cases there are UCDs that are obviously applicable to groups.
As an example, a group representing a pair of equatorial coordinates could
have the UCD "pos.eq" in addition to its two members having UCDs "pos.eq.ra"
and "pos.eq.dec".
However, this is more the exception than the rule and for the most part
there won't be much actionable metadata conveyable by group-level UCDs.

Special types of groups
-----------------------

The VOTable standard defines a few dedicated group names with specific
semantics: a, b, and c.
These may be useful to us, and it appears that we could generate at least
the primary-key and foreign-key ones from existing Felis entities.

The row-ordering group could be defined using the generic Felis group
support proposed below, but it may be more consistent with the Felis philosophy
to include it more explicitly as part of the core Felis data model.
This requires additional thought.
How the row-ordering group might be used in the RSP is addressed in several
places below.

Supporting Groups in Felis
==========================

It appears straightforward to extend the core Felis Pydantic metamodel
to allow for the formation of groups on sets of references to columns.

(example)

Linking groups to documentation
-------------------------------

(outline the problem; the TAP_SCHEMA-based documentation system doesn't work unless we adopt the IRSA groups-table model)

Annotating VOTable Results with Groups
======================================

Annotating TAP results
----------------------

IRSA TAP's group metadata model
-------------------------------

Making groups usable in other VOTable producers
-----------------------------------------------

Exploiting Groups in the RSP Portal
===================================

Using groups in result displays
-------------------------------

* table ordering

Using groups in query-building
------------------------------

Group discovery
---------------

Exploiting Groups in Python
===========================

Using groups in Parquet file access
-----------------------------------

Reading tabular data from Parquet files can be an order of magnitude or more
efficient than from other tabular data formats, because of the column-store
nature of Parquet, in which I/O is only performed for the subset of columns
that is of interest at any given time, and because the column orientation of
data storage allows (typically) more efficient data compression than row-wise
storage permits.

One of the desiderata for a Python interface to column-group metadata would
be to allow the selection of columns to be read from a Parquet file to be
specified by group in addition to by individual columns.
For example, "read all the ``cModel`` flux columns from the ``Object`` table".
