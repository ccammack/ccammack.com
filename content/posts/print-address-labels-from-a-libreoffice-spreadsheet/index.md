---
title: "Print Address Labels From a LibreOffice Spreadsheet (Mail Merge)"
date: 2018-01-27T02:20:14-07:00
tags: ["LibreOffice"]
---

I only need to do this once a year, and every year I forget how. MailMerge in [LibreOffice](https://www.libreoffice.org/) is somewhat [fiddly](http://abhweb.org/jima/mailing-list-labels-in-open-office.php) and the recommended method seems to change from version to version. These instructions work with **version 5.4.5.1**.

<!--more-->

1. Begin with a spreadsheet of addresses. My example file is called *addresses.csv* and contains columns for **first**, **last**, **address**, **city**, **state** and **zip**.

	The exact structure of the spreadsheet doesn't matter, but using fewer columns makes layout easier, so one could also use a single column for **full name** and a single column for **city, state and zip** if that makes more sense.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/address-spreadsheet-original.png" caption="Sample address spreadsheet in LibreOffice Calc">}}

1. Create a *temporary folder* to hold the working files, which can all be deleted afterwards. In my case, the folder is called *temp* and it was created on the desktop.

1. Close the original address spreadsheet if it's currently open. Make a *temporary* working copy and name the copy using a **single-character** file name.
I my case, I made a copy of **addresses.csv**, named it **a.csv**, and moved it into the temporary folder.

	> These steps lay out the address fields on a visual representaton of the label inside LibreOffice. Since file and field names can potentially be longer than the values they contain, using long names can overflow the available label area and confuse the layout. Use single-character names to ensure good results when laying out the label structure.

1. Open the temporary working copy (**a.csv**) of the address spreadsheet. **Row 1** must be a header row that specifies names for each column, so add that row if needed.
Again, use *single-character names* for the header row. In the example file, I have renamed the original columns to **f**, **l**, **a**, **c,** **s** and **z**.
Also rename the spreadsheet tab (at the bottom) to a *single-character name*, such as **a**.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/address-spreadsheet-working-copy.png" caption="Working copy of the address spreadsheet with short names">}}

1. Create a *temporary* database from the spreadsheet by selecting **File>New>Database**, then select **Connect to an existing database**, specify **Spreadsheet** in the dropdown and press the **Next** button.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/database-wizard-1.png" caption="Database Wizard Step 1: Select database">}}

1. Press the **Browse** button, select the working copy of the spreadsheet file (**a.csv**) and press the **Next** button.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/database-wizard-2.png" caption="Database Wizard Step 2: Set up spreadsheet connection">}}

1. Select **Yes, register the database for me**, then **disable** the checkbox that says **Open the database for editing** and press the **Finish** button.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/database-wizard-3.png" caption="Database Wizard Step 3: Save and proceed">}}

1. Save the new database file to the temporary folder. Again, use a short name such as **d.odb** for the file name.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/save-database-file.png" alt="Save the new database file to the temporary folder">}}

1. Select **File>New>Labels** to open the **Labels** dialog.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/menu-file-new-labels.png" alt="Select File>New>Labels to open the Labels dialog">}}

1. Select the **Labels** tab at the top of the dialog and set the **Format**, **Brand** and **Type** fields according to the label manufacturer's recommendations.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/labels-dialog-labels-tab.png" alt="Labels tab: Set format, brand and type fields">}}

1. Select the **Options** tab at the top of the dialog, then enable the **Synchronize contents** checkbox and press the **New Document** button.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/labels-dialog-press-new-document-button.png" alt="Options tab: Enable the Synchronize contents checkbox and press the New Document button">}}

1. LibreOffice Writer will open a new *Untitled* document showing blank labels laid out on the page along with a floating **Synchronize** dialog.
Drag the **Synchronize** dialog away from the top-left label to make some room if needed.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/writer-blank-label-document.png" alt="LibreOffice Writer document containing an array of blank labels">}}

1. **Click the top-left empty label** to give it focus and select **Insert>Frame>Frame...** to open the **Frame** dialog.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/writer-insert-frame.png" alt="Open the Frame dialog">}}

1. Select the **Type** tab at the top of the **Frame** dialog and adjust the settings:
	* Enable both of the the **AutoSize** checkboxes for **Width** and **Height**
	* Select **As character** for the **Anchor** setting
	* Select **Center** for the **Vertical** position setting
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/frame-dialog-type-tab.png" alt="Frame Dialog: Type tab settings">}}

2. Select the **Borders** tab  at the top of the **Frame** dialog and adjust the settings:
	* Select the **Set No Borders** preset in the **Line Arrangement** setting
	* Press the [**OK**] button to insert the new frame into the top-left label
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/frame-dialog-borders-tab.png" alt="Frame Dialog: Borders tab settings">}}

1. Use the mouse to drag the new frame's green handles and expand the frame until it is approximately the same size as the label.
	{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/expand-frame.png" alt="Expand the frame to match the size of the top left label">}}

1. Select **View>Data Sources** and drill down the database tree to find the temporary address database created earlier.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/writer-view-data-source.png" alt="Open the address database as a data source in LibreOffice Writer">}}

1. One at a time, drag and drop the column headings from the **Data Source** down onto the top-left label.
After dropping each one, click the mouse in the label area after the text to give it focus so you can insert spaces and punctuation.
Press **Enter** to create new lines as needed to format the address label.

	For example, to insert the **first name** onto the label:
    * drag the **f** column down to the label and release the mouse button to drop it
    * click the label after the text to give it focus
    * press the **Space bar** to insert a space after the first name
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/first-name-drag.png" caption="Drag the first name column from the data source down onto the label">}}
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/first-name-drop.png" caption="Drop the first name column onto the label">}}
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/first-name-insert-space.png" caption="Click the label after the text to give it focus and add a space">}}

1. Repeat this drag and drop process for each of the columns in the address database.
LibreOffice will not display the name of each field, which in this case means that the label will contain a collection of placeholder **d**'s,
but there are only a few fields in an address, so it's not too hard to get right.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/data-source-columns-placed-on-label.png" caption="Data source placeholders with punctuation assigned to the top left label">}}

1. After all of the database fields have been added to the label, again click the mouse at the end of the first label area to give it focus, then insert a **NextRecord** marker:
	* Select **Insert>Field>More Fields...**
	* Select the **Database** tab
	* Select **NextRecord** as the **Type**
	* Select the database (**d**) and sheet name (**a**) as the **Database selection**
	* Press the **Insert** button to insert the value **Next record:d.a**
	* Press the **Close** button to close the Fields dialog
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/insert-next-record-field.png" caption="Insert a NextRecord field at the end of the label">}}
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/insert-next-record-field-as-last-value-of-label.png" caption="Completed label with placeholder fields and a NextRecord value at the end">}}

1. By default, the label text will be left-aligned, but I think it looks better centered, so click the mouse on the label to give it focus again,
press **Ctrl+A** to select all of the label text, and press **Ctrl+E** to center it horizontally.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/horizontally-center-label-text.png" alt="Label text horizontally centered">}}

1. Press the **Synchronize Labels** button to copy the layout template from the first label to all of the other labels.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/press-synchronize-labels-button.png" alt="Press the Synchronize Labels button">}}

1. Select **File>Print** to print the labels. A warning dialog should appear to ask if **you want to print a form letter?** Press the **Yes** button to open the Mail Merge dialog.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/print-form-letter-warning.png" alt="Warning dialog: Do you want to print a form letter?">}}

1. On the Mail Merge dialog
	* Select **All** for **Records**
	* Select **File** for **Output**
	* Select **Save as single document** for **Save merged document**
	* Press the **OK** button
	{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/mail-merge.png" alt="Mail Merge dialog">}}

1. Save the label file to a temporary name such as **p.odt**.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/save-label-file.png" alt="Save the label file">}}

1. Open **p.odt** to view and print the mail-merged address labels.
{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/formatted-label-output.png" alt="Printable mail-merged address labels">}}

1. To clean up afterwards:
	* Delete the temporary work files and **temp** desktop folder created earlier
	* Unregister the temporary database created earlier by opening **Tools>Options...>LibreOffice Base>Databases**, selecting the **Registered Database** created ealier (**d**) and pressing the **Delete** button
	{{< figure src="/img/print-address-labels-from-a-libreoffice-spreadsheet/delete-temporary-database.png" alt="Unregister the temporary database">}}
