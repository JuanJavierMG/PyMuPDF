.. _Story:

================
Story
================

.. role:: htmlTag(emphasis)

* New in v1.21.0

================================= =============================================================
**Method / Attribute**            **Short Description**
================================= =============================================================
:meth:`Story.reset`               "rewind" story output to its beginning
:meth:`Story.place`               compute story content to fit in provided rectangle
:meth:`Story.draw`                write the computed content to current page
:meth:`Story.warnings`            read any warnings generated by HTML/CSS parsers
:meth:`Story.element_positions`   callback function logging currently processed story content
:attr:`Story.body`                the story's underlying :htmlTag:`body`
================================= =============================================================

**Class API**

.. class:: Story

   .. method:: __init__(self, html=None, user_css=None, em=12, archive=None)

      Create a story, optionally providing HTML and CSS source. In any case, this object points to a HTML structure, a so-called DOM (Document Object Model). This structure may be modified: content (text, images) may be added, copied, modified or removed by using methods of the :ref:`Xml` class.

      When finished, the story can be written to pages provided by a :ref:`DocumentWriter`.

      Here are some general remarks:

      * The :ref:`Story` constructor parses and validates the provided HTML and CSS sources. PyMuPDF provides a number of ways to manipulate the HTML source by providing access to the *nodes* of the underlying DOM. HTML documents can be completely built from ground up programmatically, or existing HTML can be modified pretty arbitrarily. For details of this interface, please see the :ref:`Xml` class.
      
      * If no (or no more) changes to the DOM are required, the story is ready to produce PDF pages via some :ref:`DocumentWriter`. To achieve this, the following loop should be used:
      
        1. Request a new, empty page from a :ref:`DocumentWriter`.
        
        2. Determine one or more rectangles on the page, that should receive story data. Note that not every page needs to have the same set of rectangles.
        
        3. Pass each rectangle to the story, each time checking whether the story's data is exhausted. If so, leave the loop, otherwise pass the next rectangle to it, respectively restart the loop requesting the next page.

      * Which part of the story will land on which rectangle / which page, is fully under control of the :ref:`Story` object and cannot be predicted.
      
      * Optionally, a story can pass back information to a Python callback function about page positions of HTML headers (tags :htmlTag:`h1` - :htmlTag:`h6`) and nodes with an ``"id"`` attribute. This can conveniently be used for automatic generation of a Table of Contents, an index of images or the like.

      * Images may be part of a story. They will be placed together with any surrounding text.

      * Multiple stories may -- independently from each other -- write to the same page. For example, one may have separate stories for page header, page footer, regular text, comment boxes, etc.


      :arg str html: HTML source code. If omitted, a basic minimum is generated (see below).
      :arg str user_css: CSS source code. If provided, must contain valid CSS specifications.
      :arg float em: the default text font size.
      :arg archive: an :ref:`Archive` from which to load resources for rendering. Currently supported resource types are images and text fonts. If omitted, the story will not try to look up any such data and may thus produce incomplete output.
      
         .. note:: Instead of an actual archive, valid arguments for **creating** an :ref:`Archive` can also be provided -- in which case an archive will temporarily be constructed. So, instead of ``story = fitz.Story(archive=fitz.Archive("myfolder"))``, one can also shorter write ``story = fitz.Story(archive="myfolder")``.

   .. method:: place(where)

      Calculate that part of the story's content, that will fit in the provided rectangle. The method maintains a pointer which part of the story's content has already been written and upon the next invocation resumes from that pointer's position.

      :arg rect_like where: layout the current part of the content to fit into this rectangle. This must be a sub-rectangle of the page's :ref:`MediaBox<Glossary_MediaBox>`.

      :rtype: tuple[bool, rect_like]
      :returns: a bool (int) `more` and a rectangle `filled`. If `more == 0`, all content of the story has been written, otherwise more is waiting to be written to subsequent rectangles / pages. Rectangle `filled` is the part of `where` that has actually been filled.

   .. method:: draw(dev, matrix=None)

      Write the content part prepared by :meth:`Story.place` to the page.

      :arg dev: the :ref:`Device` created by `dev = writer.begin_page(mediabox)`. The device knows how to call all MuPDF functions needed to write the content.
      :arg matrix_like matrix: a matrix for transforming content when writing to the page. An example may be writing rotated text. The default means no transformation (i.e. the :ref:`Identity` matrix).

   .. method:: element_positions(function, args=None)

      Let the Story provide positioning information about certain HTML elements once their place on the current page has been computed - i.e. invoke this method **directly after** :meth:`Story.place`.

      :arg function: a Python function which will be invoked by this method to process positioning information.
      :arg dict args: an optional dictionary with any **additional** information that you want to provide to ``function``. Like for example the current output page number. Every key in this dictionary must be a string that conforms to the rules for a valid Python identifier. The complete set of information is explained below.

   .. method:: reset()

      Rewind the story's document to the beginning for starting over its output.

   .. method:: warnings()

      An entirely optional method to check for any errors when parsing HTML / CSS source. Use this if the source quality is uncertain and / or may contain HTML / CSS language elements that are not (yet) supported by the parsers used by MuPDF. For example, MuPDF only supports CSS up to level 2, not the more advanced level 3.

      .. caution:: This method will invalidate the :ref:`Story` object! The same story must be created again afterwards -- so the method should only be used if there is a high chance to encounter problematic sources.

      :returns: a string with error messages generated by the source parsing.

   .. attribute:: body

      The :htmlTag:`body` part of the story's DOM. Even if `html=None` has been used at story creation, the following minimum HTML source will always be available::

        <html>
            <head></head>
            <body></body>
        </html>

      This attribute contains the :ref:`Xml` node of :htmlTag:`body`. All relevant content for PDF production is contained between "<body>" and "</body>".


Element Positioning CallBack function
--------------------------------------

The callback function can be used to log information about story output. The function's access to the information is read-only: it has no way to influence the story's output.

A typical loop for executing a story with using this method would look like this::

    HTML = """
    <html>
        <head></head>
        <body>
            <h1>Header level 1</h1>
            <h2>Header level 2</h2>
            <p>Hello MuPDF!</p>
        </body>
    </html>
    """
    MEDIABOX = fitz.paper_rect("letter")  # size of a page
    WHERE = MEDIABOX + (36, 36, -36, -36)  # leave borders of 0.5 inches
    story =  fitz.Story(html=HTML)  # make the story
    writer = fitz.DocumentWriter("test.pdf")  # make the writer
    pno = 0 # current page number
    more = 1  # will be set to 0 when done
    while more:  # loop until all story content is processed
        dev = writer.begin_page(MEDIABOX)  # make a device to write on the page
        more, filled = story.place(WHERE)  # compute content positions on page
        story.element_positions(recorder, {"page": pno})  # provide page number in addition
        story.draw(dev)
        writer.end_page()
        pno += 1  # increase page number
    writer.close()  # close output file

    def recorder(elpos):
        pass

Attributes of the Element Positions object
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The parameter passed to the ``recorder`` function is an object with the following attributes:

* ``elpos.depth`` (int) -- depth of this element in the box structure.

* ``elpos.heading`` (int) -- the header level, 0 if no header, 1-6 for :htmlTag:`h1` - :htmlTag:`h6`.

* ``elpos.id`` (str) -- value of the ``id`` field, or "" if n/a.

* ``elpos.rect`` (tuple) -- element position on page.

* ``elpos.text`` (str) -- immediate text of the element.

* ``elpos.open_close`` (int bit field) -- bit 0 set: opens element, bit 1 set: closes element. Relevant for elements that may contain other elements and thus may not immediately be closed after being created / opened.

* ``elpos.rect_num`` (int) -- count of rectangles filled by the story so far.

* ``elpos.page`` -- any additional, user-supplied information, like in this case the page number.

