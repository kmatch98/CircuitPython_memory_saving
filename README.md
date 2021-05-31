# CircuitPython solutions to running out of Memory <br/>especially text and graphics

CircuitPython boards have two types of memory, non-volatile memory where programs are stored, and volatile-RAM where your active variables are kept. Non-volatile memory is for long-term storage of your `code.py` file and all library files. You may also have bitmaps and font files stored in non-volatile memory, sometimes called EEPROM. If you run out of non-volatile memory, you usually observe an error when trying to copy a file onto the `CIRCUITPY` drive, giving an error like “out of space, cannot save file”. 

If you can’t save a file to the CIRCUITPY drive, the solutions are to remove other unneeded files or find an alternate board that has a larger non-volatile storage.  Or, you could find a board that utilizes an SD card and can pull files from there. Here’s a learning guide on using an SD card with CircuitPython: https://learn.adafruit.com/micropython-hardware-sd-cards?view=all.  If you are using font files, shrink the size of them by deleting unnecessary glyphs.  [This is easy for BDF files since they can be edited directly with a text editor.](https://learn.adafruit.com/custom-fonts-for-pyportal-circuitpython-display/conversion)

When running CircuitPython programs, they use volatile memory (often called RAM, random access memory) to store all your active variables.  This includes any graphics files (bitmaps and fonts) that you load into RAM for displaying onto an attached screen. Each board has a different amount of RAM so look at the product details to see what your board has available. Some of the board’s RAM will be used by the CircuitPython system so not all will be available to you for your variables. 

### `Memory Allocation Failed` errors

Most basic programs won’t have any issues, but if your program uses a lot of memory you may encounter an error like this:

```python
  File "code.py", line 12, in <module>
  File "adafruit_bme280.py", line 445, in __init__
MemoryError: memory allocation failed, allocating 158 bytes
```

This error message describes the trail of code that led to this error.  There are a couple of tools to assess what’s using up your non-volatile RAM. There is a library called `gc` which stands for “garbage collector” that has a function called `gc.mem_free()` used to monitor the amount of available RAM.  You can use the following to print the available RAM used at different locations throughout your program:

```python
import gc

# a lot of code here

gc.collect()
print(“Free memory at code point 1: {} bytes”.format(gc.mem_free()) )
```

Sprinkle these throughout your code to see how much RAM you have available at different locations and identify the steps that are using the most RAM. Be sure to include `gc.collect()` prior to printing the amount of memory free. We’ll explain later why. 

You can monitor the memory to see which steps in your code are using up
the memory. This is a good tool to identify where you should focus your effort for memory optimization. If you are loading a lot of fonts or bitmap files (using the Adafruit_Imageload library) you can run out of RAM in a hurry. By printing gc.mem_free() you can diagnose the biggest memory users and focus on them first. Also you may consider measuring the memory use of your imported libraries to see if they are significant, just put a gc.collect() and then print the gc.mem_free() both before and after each import statement. 

### Minimize imports
If an import is using a lot of memory, you can selectively import only a subset of a library’s functions. 

```python
from adafruit_display_text import bitmap_label.label as Label
```

This imports only the Label function from the adafruit_display_text library. The actual impact will depend on exactly how the library is created.  Try it out and see if you can reduce the memory used by your imports by selectively importing the functions you are using.

### Biggest memory user #1: Bitmaps

If you are using Adafruit_Imageload for displaying bitmaps, that may be a large user
of your precious RAM. Regarding bitmaps, one memory-saving alternative to Adafruit_Imageload is to [display directly from the stored file using the OnDiskBitmap functions](https://learn.adafruit.com/circuitpython-display-support-using-displayio/display-a-bitmap#ondiskbitmap-3026044-6)

Using OnDiskBitmap does not store the bitmap in RAM, it just draws it directly from the stored file location (could be the CIRCUITPY drive or an SD memory card). The downside is that the display will not update as fast when using OnDiskBitmap since it has to be loaded from the non-volatile memory which is often slower, and OnDiskBitmap does not take advantage of displayio’s “dirty rectangle” tracking that reduces redraw times. 

If you need the fast display redrawing that you get with Adafruit_Imageload of bitmaps, you can consider simplifying the color depth of your bitmaps. The bitmaps that CircuitPython can use are so-called “indexed” bitmaps. They contain a palette of colors used in the bitmap, and then each pixel on the bitmap has an entry in the file that tells it which palette color should be used. If you can reduce the number of colors then it may reduce memory usage significantly. But keep in mind due to the binary nature of how bitmap colors are stored, there are breakpoints in the number of colors that will have an impact on memory usage. For example, if you have 32 colors, going to 31 colors won’t save anything, but reducing to 16 colors will practically halve the amount of RAM that the bitmap uses. You’ll need an external image editor to change the amount of colors, you can probably find tools online that can do this. The main jumps occur between 2, 4, 8, 16, 32, 64, 128, and 256 (they’re binary factors of 2).
Try reducing your color depth by a factor of two and see how much it helps and whether that is a good solution for your program and what you want to display. 

Get rid of bitmaps.  If your bitmaps are relatively simple shapes and lines, don’t use bitmaps, but instead use the `vectorio` functions. You can draw lines, circles and polygons and it doesn’t use much RAM at all. If at all possible, use `vectorio` shapes to conserve RAM and ditch the bitmaps. [Learn more about vectorio in the docs.](https://circuitpython.readthedocs.io/en/latest/shared-bindings/vectorio/index.html)

### Biggest memory users #2: Fonts and Text Labels
Fonts can also take up a lot of RAM since they are basically a collection of little bitmaps for each character glyph. If you need to display different font sizes, consider loading one smaller font file and then use the `scale` parameter in the `label` or `bitmap_label` from the display_text library. 

If you’re going to use text labels with displayio, start with `bitmap_label`, it uses less RAM than `label`. (There are some counterintuitive situations where you may want to use `label` even though it uses more total RAM, we’ll discuss that situation later.)

- use bitmap_label for creating text labels (library adafruit_display_text)
- Reduce the color depth of bitmaps
- Load bitmaps directly from non-volatile memory using OnDiskBitmap
- Load one small font and scale it as needed (see scale parameter for labels)
- Use vectorio for graphics whenever possible instead of bitmaps.

### Memory allocation failed, but I have plenty of memory free! (memory fragmentation)
Sometimes, you can see with gc.mem_free() that you have plenty of memory available, but you still get a message about “Memory allocation failed”. Ok now we dig deeper into how CircuitPython manages memory with the garbage collector. 

CircuitPython manages RAM by cleaning up unused variables with the garbage collector. Whenever a memory object is created, it is placed at the next available memory location that can fit the memory object. The garbage collector’s job is to “clean up” and remove any phantom unused memory objects by using the gc.collect() command. This command will get automatically triggered sometimes, or you can call it in your code directly. As an example of the garbage collector, after a function finishes, all its local variables are automatically no longer used so the garbage collector could go free up all that memory space that was previously allocated in that function. However, the garbage collector is only automatically triggered when a memory allocation requests a chunk larger than any available memory space. For this reason it’s best to sprinkle in gc.collect() in your code to help reduce memory fragmentation. After a function returns it is a good time to call gc.collect() so you can free up space.  Another good time is just prior to creating large objects. There could be “unused” phantom variables clogging up a space that could accommodate your new big variable. By running the garbage collector, you clear out those unused variables so the new memory allocation can be placed in that space. 

With the memory organization structure in CircuitPython, it can cause the memory to become fragmented, meaning that after running for a while large contiguous chunks of free memory are not available. When you create a memory object, it requires a contiguous block of RAM to be allocated. If such a sized block isn’t not available, an error is raised “Memory allocation failed”.   So, you can have have plenty of RAM free, but if a large enough chunk is not available you can trigger an error. A common observed occurrence is when loading a bitmap or creating a large text with `bitmap_label`, two situations that can request an object requiring a decent amount of RAM. (Note: Doing a “defragmentation” of memory is not feasible with the current memory structure of CircuitPython, but we give you some ways to help prevent that from happening).  So the following suggestions are ways to reduce your memory fragmentation.  

Other note: some chips don’t have much RAM, like the SAMD21 series.  To use these may require more work to achieve your goals, or you could consider another chip with a bit more RAM for your needs. 

Reducing memory fragmentation:
- For bitmaps and text that are static through your program, allocate them early in your code. If you allocate them toward the end, it is a higher chance that your memory will get fragmented and cause a memory allocation error when you later create the bitmap or text label. Allocate these large items early while memory space is relatively wide open. 
- Special note with text labels: bitmap_label uses less overall RAM but it needs it all in one large chunk. Somewhat counterintuitively, label uses more total RAM but splits it up into smaller pieces, so consider using label if you need to allocate text labels with often changing text.  This is one way to “walk between the raindrops” when your memory is fragmented. Also, remember if you redefine the text in a bitmap_label, it reallocates the whole bitmap, requiring a new chunk of RAM. If you really need to redo the text in a label frequently, evaluate whether `label` can help.
- Use functions where it makes sense for memory items that are temporary. Take advantage of the fact that after a function closes, all its local variables are automatically “unused”. Call gc.collect() after function returns to reclaim all that memory space as free again.
- Use gc.collect() prior to any large memory allocations.  This will help reduce the fragmentation since you clear out all phantom unused variables _before_ you request a new chunk. 
- Advanced programmers: allocate a large memory buffer early in the life of your code and reuse the same memory buffer through your program.

### Other memory conserving tips:
- Be sure all your libraries are pre-compiled as .mpy files. if you pull libraries from thr CircuitPython bundle, they are already pre-compiled to .mpy files. [Go here to learn to compile python code to .mpy files.](https://learn.adafruit.com/welcome-to-circuitpython/frequently-asked-questions#how-can-i-create-my-own-mpy-files-3020687-11)
- Rather than keeping a big list use list generators, e.g. range() 
- Try to import big modules first.
- Take ou the library module file from the directory package and use it directly at the ``lib`` level.

### Other memory-related issues:
- “pystack exhausted” errors - you may be doing something with a lot of of levels of function calls or recursion. Maybe try another way of solving your problem. Other times the cause is unclear, ask for guidance on discord: https://adafru.it/discord

### Other resources on CircuitPython memory usage:

- FAQ: See “Memory” in: https://learn.adafruit.com/welcome-to-circuitpython/frequently-asked-questions

- FAQ: Deeper dive via microPython and very relevant to CircuitPython: https://docs.micropython.org/en/latest/reference/constrained.html

- Video: Project specific - @foamyguy's livestream on saving memory on the EZbake oven for the big-screened PyPortal Titano: https://youtu.be/lDAyfZp_350

- Video: Deep dive with Scott Shawcroft - into the weeds of CircuitPython memory use: https://youtu.be/baa5ILZTRkQ

