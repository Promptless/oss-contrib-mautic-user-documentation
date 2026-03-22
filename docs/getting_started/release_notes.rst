.. vale off

Release notes
#############

.. vale on

This page documents notable changes, bug fixes, and improvements in Mautic releases.

Mautic 7.x
**********

Bug fixes
=========

Landing Page Builder code editor fix
------------------------------------

Fixed an issue where the GrapesJS Builder's code editor duplicated ``<head>`` content into the ``<body>`` section when saving Landing Pages.

Previously, when editing HTML in the Landing Page Builder using the 'Edit code' feature, each save caused ``<head>`` content such as ``<meta>`` tags and ``<title>`` to appear duplicated in the ``<body>`` section. The duplication compounded with each save operation.

This fix ensures that the code editor properly handles HTML content in Landing Pages, preserving the correct document structure when saving.

.. note::
    This fix applies only to Landing Pages edited in HTML mode within the GrapesJS Builder. MJML mode for Emails wasn't affected by this issue.

For more information about using the Email and Landing Page Builder, see :doc:`/builders/email_landing_page`.
