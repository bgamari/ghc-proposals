The GHC Proposals Process
=========================

GHC proposals move through a set of phases,

- *Development*: Develop a proposal document laying out the motivation for your
  proposed change, specifying the change, and assessing its benefits
  and drawbacks. Upload this document to the GHC's proposal repository as a pull
  request.

- *Discussion*: The community will debate the pros and cons of your proposal
  as described in the document. You may edit and improve your proposal document at any time. This
  debate phase is limited to 4 weeks after the last edit.

- *Committee decision*: The GHC Steering Committee will review your proposal
  document and its ensuing debate and decide whether to accept or reject
  your proposal. If rejected you may amend and resubmit your proposal document
  at any time.

- *Implementation*: Accepted proposals may be implemented and submitted for code review.
  In most cases, proposal authors will implement the change themselves, but there is no obligation to do so.


Technical Facilitation of the Process
-------------------------------------

We are using GitHub's pull request infrastructure to facilitate proposal
collection and discussion. 

Pull request tags are used to communicate in which phase the proposal
is currently in.

- While still under development, please tag your proposal with ``development``.
- If you want to open your proposal for debate, tag it with ``debate``.
- Give ample time to hear opinions as this will strengthen your case. When you
  are ready to receive an accept/reject of the GHC Steering committee, tag it
  with ``@ghc-steering``
- Accepted proposals will be marked as ``@>->-`` by the Steering committee.

Content of the Proposal Document
--------------------------------

Each proposal document must follow the following outline (Template provided in `proposal template <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0000-template.rst>`_.)

a. *Motivation*: Give a strong reason for why the community needs this change. Describe the use case as clearly as possible and give an example. Explain how the status quo is insufficient or not ideal.

b. *Proposed Change Specification*: Specify the change in precise, comprehensive yet concise language. Strive for a complete definition. Your specification may include,

- grammar and semantics of any new syntactic constructs
- the types and semantics of any new library interfaces
- how the proposed change addresses the original problem (perhaps referring back to the example)
- how the proposed change might interact with existing language or compiler features

c. *Effect and Interactions*. Detail how the proposed change addresses the original problem raised in the motivation. Detail how the proposed change interacts with existing language or compiler features and provide arguments why this is not going to pose any problems.

d. *Costs and Drawbacks*. Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.

e. *Alternatives*: List existing alternatives to your proposed change as they currently exist and discuss why they are insufficient.

f. *Unresolved questions*: Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

g. *Implementation Plan* (Optional): If accepted who will implement the change? Which other ressources and prerequisites are required for implementation? 


Note that proposals are written in `ReStructuredText
<http://www.sphinx-doc.org/en/stable/rest.html>`_ rather than Markdown for its
expressiveness and ease of integration into other GHC infrastructure. See the
`GHC Users Guide
<http://downloads.haskell.org/~ghc/latest/docs/html/users_guide/editing-guide.html>`_
for a brief introduction to ReStructuredText.les introduced in the *Motivation*
section).


Technical Issues of Proposal Submission
---------------------------------------


To begin a new proposal copy the `proposal template <https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0000-template.rst>`_. with a new name briefly describing your
proposal (e.g. ``0000-type-indexed-typeable.rst``). 

This can be done either online
through GitHub's in-place editing feature (the pencil icon visible when viewing
a file on GitHub) or locally by forking this repository and cloning the fork.
-- the former will not work -- 


See GitHub's `documentation <https://help.github.com/articles/fork-a-repo/>`_ if
you are unfamiliar with this aspect of GitHub's workflow.


Note that proposals are written in `ReStructuredText
<http://www.sphinx-doc.org/en/stable/rest.html>`_ rather than Markdown for its
expressiveness and ease of integration into other GHC infrastructure. See the
`GHC Users Guide
<http://downloads.haskell.org/~ghc/latest/docs/html/users_guide/editing-guide.html>`_
for a brief introduction to ReStructuredText.


Submitting your proposal for discussion
---------------------------------------

The discussion of your proposal will take place on a pull request of the
``ghc-proposals/ghc-proposals``
`repository <https://github.com/ghc-proposals/ghc-proposals>`_.

In order to open your proposal document for the debate phase,
push your proposal to your ``ghc-proposals`` fork and open a pull
request against the ``master`` branch of ``ghc-proposals/ghc-proposals``. If you
unfamiliar with GitHub pull requests then see the
`relevant documentation <https://help.github.com/articles/creating-a-pull-request/#creating-the-pull-request>`_.

The pull request description should include a brief description of your
proposal, along with a link to the rendered proposal document. For instance, it
might look like,

.. code-block::

    This is a proposal augmenting our existing `Typeable` mechanism with a
    variant, `Type.Reflection`, which provides a more strongly typed variant as
    originally described in [A Reflection on
    Types](http://research.microsoft.com/en-us/um/people/simonpj/papers/haskell-dynamic/index.htm)
    (Peyton Jones, _et al._ 2016).

    [Rendered](https://github.com/bgamari/ghc-proposals/blob/typeable/proposals/0000-type-indexed-typeable.rst)


What happens next
-----------------

After you submit your proposal docment, the community is invited to comment and
debate. Feel free to improve your document to reflect the discussion. The goal
is to make the strongest case possible and demonstrate that all alternatives
have been considered. It is very likely that multiple iterations are necessary
before the proposal is ready for review.

If you think the proposal is ready for review by the GHC steering committee move
to the next phase (see Technical Facilitation of the Process, above).

For more details see (Link to the main document)

----- SNIP ----

The development phase
---------------------

The development phase begins after a pull request has been opened for your
proposal. This is the phase where the Haskell community discusses the merits and
possible shortcomings of the proposal.

Proposals which are in this phase should be tagged with the
``development`` tag.





5. Discussion will proceed on the pull request; it is very likely that multiple
   iterations will be necessary before the proposal stabilizes.

6. When discussion has died down notify the `GHC Commitee
   <steering-committee.rst>`_ with a comment mentioning ``@bgamari``. The
   committee will review the proposal, as well as the feedback collected on the
   pull request and general community sentiment, and decide whether the proposal
   will be accepted.

. When your proposal is accepted your pull request will be merged. At this
   point you or someone else may choose to implement your proposal.
