::

    Status: Active
    Type: Process
    Created: 2020-02-04
    Authors: Łukasz Langa <lukasz@edgedb.com>
    RFC PR: `edgedb/rfcs#0003 <https://github.com/edgedb/rfcs/pull/3>`_

=========================
RFC 0001: The RFC Process
=========================

This RFC describes the process of submitting, discussing, and deciding
on large-scale changes to EdgeDB.  It is loosely modeled after Python's
`PEP process <https://www.python.org/dev/peps/pep-0001/>`_ which in turn
is loosely modeled after the `Request For Comments
<https://en.wikipedia.org/wiki/Request_for_Comments>`_ process dating
back to the Spring of '69.


When to write an RFC
====================

The "Request For Comments" document is intended to be the primary
mechanism for proposing a major change to EdgeDB, both the database
server, as well as related technology like the query languages, binary
protocols, client APIs, and so on.

The primary audience for RFCs are the core developers of EdgeDB, as well
as the community whose input on the related issue is collected.

The EdgeDB user community may choose to use the process to design and
document expected conventions, integrations, manage complex design
coordination problems that require collaboration across multiple
projects.

The following non-exhaustive list demonstrates the kinds of changes
that should be discussed through an RFC:

* new major feature of EdgeDB (access control, database migrations,
  automatic admin panel, and so on);

* incompatible changes in any API, including the command line;

* new or changed query syntax.

For changes of smaller scale use the EdgeDB issue tracker or another
related issue tracker.


Structure of an RFC
===================

Document Format
---------------

An EdgeDB RFC document is a UTF-8 encoded text file formatted with
`ReStructured Text <https://docutils.sourceforge.io/rst.html>`_.
Therefore, it uses the `.rst` file extension.

The file name should start with a unique four digit number, left-padded
with zeroes if necessary, followed by a lowercase
`slug <https://docs.djangoproject.com/en/3.0/glossary/#term-slug>`_.

Lines in RFC documents should be wrapped at 72 characters.  Lines are
ended with Unix-style newline characters.  Trailing whitespace should
be removed from all lines.  The last line should end with a newline
character.  You can use the ``.editorconfig`` file in this repository
to help you with those requirements.

Required Sections
-----------------

1. An `IETF RFC 822 <https://tools.ietf.org/html/rfc822>`_ **Preamble**
   with metadata about the RFC.  See the top of this RFC for a list of
   accepted fields and their format.

2. A short, ideally tweet-size, **abstract** describing of the issue
   being addressed.

3. A short but crucial section on **motivation** behind the change.  It
   should clearly explain why the existing situation is inadequate to
   address the problem that the RFC solves.  Submissions without
   sufficient motivation are unlikely to be discussed and ultimately
   accepted.

4. A technical **specification** of the semantics and syntax of the
   feature.  The specification should be detailed enough to allow
   competing, interoperable implementations.

5. An explicit discussion of **backwards compatibility** and
   **security implications** of the discussed change.  Both are required
   to ensure both the PEP author as well as all readers are aware of the
   consequences of the change.

6. An extensive discussion of **rejected alternative ideas**.  This
   covers both large-scale alternative approaches to the problem, as
   well as rejected details of implementation of the approach that the
   RFC is proposing.  The summary of a rejected idea should be presented
   along with the reasoning as to why it was rejected.

Optional Sections
-----------------

The RFC document should be split into sections in a natural fashion that
facilitates understanding the subject matter by reading it from top to
bottom.

Sometimes a larger section on the **rationale** behind the selected
design will be useful.  Sometimes it might make sense to discuss
**how the new feature should be taught** to newcomers or discovered by
existing users.  A discussion of the **reference implementation** might
be necessary if its technical details are the main subject matter of
the RFC.

Finally, depending on the state of the RFC, enumerating **open issues**
and **external references** might be helpful to the reader.

RFC Types and Statuses
----------------------

An RFC can describe a **Feature**, a **Process**, or a **Guideline**.
All RFCs start in the **Draft** status.  Depending on the result of
discussion, they get **Accepted**, **Rejected**, or **Deferred**.

Accepted features become **Final** once they are implemented, with
a note on which EdgeDB version the change is implemented in.

Accepted processes and guidelines become **Active**, with a note about
scope and effective date.  Later, they might become **Inactive** if they
are abandoned or replaced by a different process or guideline.


Acceptance Workflow
===================

Before you submit an RFC
------------------------

The change you have in mind might have been discussed elsewhere, or
might be part of a larger change, or might be something that is out
of scope for EdgeDB.  Before you begin writing a formal RFC, reach out
to the team on any community support channel, like our `Spectrum chat
<https://spectrum.chat/edgedb/>`_, to gather some initial feedback on
your idea.

Submitting an RFC
-----------------

RFCs are submitted to the https://github.com/edgedb/rfcs repository in
the form of pull requests.  Choose the next available number above 1000.
Lower numbers are reserved for process-related documentation and guides.
Avoid choosing "cute" arbitrary numbers for RFCs.

The RFC should have a main champion, typically the author, who is
responsible for moving the discussion forward, as well as gathering and
documenting community and core developer feedback.  Having a quick
feedback loop and an up-to-date RFC document is very helpful.

It's okay if no core developers are co-authors on a given RFC.  In this
case prepare for a few more rounds of pull request review about the
logistics of the RFC process and expected content.  Try for the initial
pull request to be as close to the document format described in the
section above.

The discussion period
---------------------

The goal of discussing the RFC is to build consensus.

Ideally the number of communication channels involved in discussing an
RFC is kept at a minimum.  The initial version will be discussed on the
original pull request.  Subsequent commits on this pull request should
represent granular changes to facilitate easy review.  Ideally they
should mark particular comments on the previous version as resolved.
Adding an idea to the "Rejected ideas" section is a form of resolution.

If there *are* external discussion channels, the RFC champion is
expected to follow them and to gather and integrate feedback from them.

All community members must be enabled to share feedback.  Moderators of
official EdgeDB communication channels enforce the Code of Conduct first
and foremost, to ensure healthy interaction between all interested
parties.  If necessary, enforcement can result in a given participant
being excluded from further discussion and thus the decision process.

Final comment period
--------------------

At some point, when the discussion no longer yields new view points,
issues, or solutions, the RFC champion or one of the core developers
can propose a "motion for final comment period", along with
a recommendation to either:

* accept;
* reject; or
* defer the RFC.

To enter the final comment period, the motion should be accompanied with
a summary comment of the current state of discussion, ideally already
represented in the RFC text.  It's especially important to include any
major points of disagreement and tradeoffs.

The final comment period lasts for ten business days to allow
stakeholders to file any final objections before a decision is reached.

Revisiting deferred and rejected RFCs
-------------------------------------

Before attempting to restart discussion of a deferred or rejected RFCs,
the relevant interested parties must contact the previous champion and
core developers active in that discussion.  If they agree there is
substantial evidence to justify revisiting the idea, a pull request
editing the deferred or rejected RFC can be opened.

Failure to get proper buy-in beforehand will likely result in immediate
rejection of a pull request on a deferred or rejected RFC.
