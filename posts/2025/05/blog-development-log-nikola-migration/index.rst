.. title: Blog Development Log (1) - Migrate to Nikola for
.. slug: blog-development-log-nikola-migration
.. date: 2025-05-18T17:00:00.000+08:00
.. tags: DevLog,NikolaBlog
.. link:
.. description:
.. type: text
.. status: draft

Here are some notes about migration process, and.

.. TEASER_END

Before the Migration
====================

The original was a bit wild -- setting up server that serves blog content with some of the contents
like post list and article dynamically rendered as requested.

The project covered from previous work experience, and it ends up with `Django <https://www.djangoproject.com>`_
as a backend service that manages posts and image assets, and a `express <https://expressjs.com>`_ service that provides
web content and renderings. Plus a website-based editor that allows me to preview.

provided, including:

* Managing a web service on cloud service provider (AWS)
* Configuration of web server proxy like `Nginx <https://nginx.org>`_
* Manage static assets like images, fonts, and favicon
* Get familiar with frontend assets bundler like `webpack <https://webpack.js.org>`_ and `Parcel <https://parceljs.org>`_

And the whole job involves things that gets more complicated than estimated:

* Design mechanism that enables the editor to renders preview in realtime, interacts with local file system, bundle the post and image into somethings , etc.

Adding additional to markdown parser
Twisting code rendering library to support layout, code copying, etc.

Pick the tools
==============

Did not took much time making decision. Blog instead of fancy profile site, so I took a peek on top of `Jamstack <https://jamstack.org/generators/>`_'s list, and chose a blog-oriented builder. It also provides community support on Docker service, because at the time of writing I'm having trouble with building runtime environment and libraries on my MacOS machine (which is also part of the reason why not choosing popular choices like `Jekyll <https://jekyllrb.com>`_).


Initial Attempts
================

TBD


Time to Turn off the Machine
============================

* Turn off VCP networks
* Turn off and delete EC2 instance that runs the server
* Update Route 53 DNS mappings from internal network to GitHub's domain
* Set up custom domain for GitHub Page so that GitHub will route the custom domain (pyhsieh.net) to (pykenny.github.io).


Some Pending Works
==================

* Check out between
* Organize CSS rules because
* Layout CSS for the post content (currently using)
* More options in
* Implement some plugins, such as calendar view


