---
title: "PHP: SortingIterator"
date: 2008-08-09
---
I just had the need for an Iterator that can sort itself by a user defined callback.

My special use case is that the DirectoryIterator of PHP does not sort the file list so it's pretty random. But my program logic relies on files being sorted by filename.

So here's the little class:

    
    class SortingIterator implements IteratorAggregate
    {
    
            private $iterator = null;
    
            public function __construct(Traversable $iterator, $callback)
            {
                    if (!is_callable($callback)) {
                            throw new InvalidArgumentException('Given callback is not callable!');
                    }
    
                    $array = iterator_to_array($iterator);
                    usort($array, $callback);
                    $this->iterator = new ArrayIterator($array);
            }
    
    
            public function getIterator()
            {
                    return $this->iterator;
            }
    }
    

Basically it uses the functionality of the function "iterator\_to\_array()" to convert the iterator into an array and then let the array getting sorted by usort() and the user-defined callback.

Then the sorted array is wrapped into an ArrayIterator so it can be used with all Iterator functions, decorators and whatever again.

**Usage example:**

    
    function mysort($a, $b)
    {
            return $a->getPathname() > $b->getPathname();
    }
    
    $it = new SortingIterator(new RecursiveIteratorIterator(new RecursiveDirectoryIterator('/home/soenke/muell')), 'mysort');
    
    foreach ($it as $f) {
            echo $f->getPathname() . "\n";
    }
    

**Without sorting Iterator:**

    
    /home/soenke/tests/test/a.txt
    /home/soenke/tests/test/b.txt
    /home/soenke/tests/test/ä.txt
    /home/soenke/tests/test/aaa.txt
    /home/soenke/tests/test/abc.txt
    /home/soenke/tests/test/az.txt
    

**And with Sorting-Iterator:**

    
    soenke@turingmachine:~/tests$ php dirit.php
    /home/soenke/tests/test/a.txt
    /home/soenke/tests/test/aaa.txt
    /home/soenke/tests/test/abc.txt
    /home/soenke/tests/test/az.txt
    /home/soenke/tests/test/b.txt
    /home/soenke/tests/test/ä.txt
    
