.. _alignments:

***************
Read alignments
***************

.. currentmodule:: HTSeq

.. doctest:: 
   :hide:

   >>> import os
   >>> if not os.getcwd().endswith('example_data'):
   ...     os.chdir('example_data')
   >>> import HTSeq

Concepts
========

There are a large number of different tools to align short reads to a reference. Most of
them use their own output format, even though the `SAM format`_ seems to become the common 
standard now that many of the newer tools use it.

.. _`SAM format`: http://samtools.sourceforge.net/SAM1.pdf

HTSeq aims to offer a uniform way to analyse alignments from different tools. To this end,
for all supported alignment formats a parse class is offered that reads an alignment file
and generates an iterator over the individual alignment records. These are represented as
objects of a sub-class of :class:`Alignment` and hence all offer a common interface.

So, you can easily write code that should work for all aligner formats. As a simple example,
consider this function that counts the number of reads falling on each chromosome:

.. doctest::

   >>> import collections
   >>> def count_in_chroms( alignments ):
   ...     counts = collections.defaultdict( lambda: 0 )
   ...     for almnt in alignments:
   ...         if almnt.aligned:
   ...             counts[ almnt.iv.chrom ] += 1
   ...     return counts

If you have a SAM file (e.g., from BWA or BowTie), you can call it with:

.. doctest::

   >>> sorted(count_in_chroms(HTSeq.BAM_Reader("yeast_RNASeq_excerpt.sam")).items()) #doctest:+NORMALIZE_WHITESPACE
   [('2-micron', 46), ('I', 362), ('II', 1724), ('III', 365), ('IV', 3015),
    ('IX', 648), ('V', 999), ('VI', 332), ('VII', 2316), ('VIII', 932),
    ('X', 1129), ('XI', 1170), ('XII', 4215), ('XIII', 1471), ('XIV', 1297),
    ('XV', 2133), ('XVI', 1509)]

If, however, you have done your alignment with Eland from the SolexaPipeline, which
uses the "Solexa export" format, you can use the same function, only using :class:`SolexaExportReader` 
instead of :class:`BAM_Reader`:

.. doctest::

   >>> count_in_chroms( HTSeq.SolexaExportReader( "mydata_export.txt" ) ) #doctest:+SKIP

Both class generate iterators of similar objects. On the other hand, some formats contain more information
and then the ``Alignment`` objects from these contain additional fields.


Parser classes
==============

Depending on the format of your alignment file, choose from the following parsers:

.. class:: BowtieReader( filename_or_sequence )
           BAM_Reader( filename_or_sequence )
           SolexaExportReader( filename_or_sequence )
           
All of these are derived from :class:`FileOrSequence`. When asked for an iterator,
they yield ``Alignment`` objects of types :class:`BowtieAlignment`, :class:`SAM_Alignment`,
or :class:`SolexaExportAlignment`. See below for their properties.
   
Adding support for a new format is very easy. Ask me if you need something and
I can probably add it right-away.
Alternatively, you can convert your format to the SAM format. The SAMtools_
contain Perl skripts to convert nearly all common formats.
   
.. _SAMtools: http://samtools.sourceforge.net/

   .. method:: BAM_Reader.peek( num = 1 ):
      
      Peek into a SAM file or connection, reporting the first ``num`` records.
      If you then call an iterator on the ``BAM_Reader``, the record will
      be yielded again.   

             
``Alignment`` and ``AlignmentWithSequenceReversal``
===================================================

.. class:: Alignment( read, iv )

   This is the base class of all Alignment classes. Any class derived 
   from ``Alignment`` has at least the following attributes:
   
   .. attribute:: read  
   
      The read. An object of type :class:`SequenceWithQuality`. See there for the sub-attributes.
      
      Note that some aligners store the reverse complement of the read if it was
      aligned to the '-' strand. In this case, the parser revers-complements the read
      again, so that you can be sure that the read is always presented as it was sequenced
      (see also :class:`AlignmentWithSequenceReversal`).
   
   .. attribute:: aligned
   
      A boolean. Some formats (e.g., those of Maq and Bowtie) contain only aligned
      reads (and the aligner collects the 
      unaligned reads in a seperate FASTQ file if requested). For these formats, ``aligned``
      is always ``True``. Other formats (e.g., SAM and Solexa Export) list all reads, including those which could
      not be aligned. In that case, check ``aligned`` to see whether the read has an
      alignment.
      
   .. attribute:: iv 
   
      An object of class :class:`GenomicInterval` or ``None``.
      
      The genomic interval to which the read was aligned (or ``None`` if ``aligned=False``).
      See :class:`GenomicInterval` for the sub-attributes. Note that different formats
      have different conventions for genomic coordinates. The parser class takes care
      of normalizing this, so that ``iv`` always adheres to the conventions outlined
      in :class:GenomicInterval. Especially, all coordinates are counted from zero, not one.

   .. attribute:: paired_end
   
      A boolean. True if the read stems from a paired-end sequencing run. (Note: At the moment
      paired-end data is only supported for the SAM format.)
      

.. class:: AlignmentWithSequenceReversal( read_as_aligned, iv )

      Some aligners store the reverse complement of the read if it was
      aligned to the '-' strand. For these aligners, the Alignment class is derived
      from ``AlignmentWithSequenceReversal``, which undoes the reverse-complement if necessary
      to ensure that the ``read`` attribute always presents the read in the ordder in which
      it was sequenced.
      
      To get better performance, this is done via lazy evaluation, i.e., the 
      reverse complement is only calculated when the ``read`` attribute is accessed 
      for the first time. The original read as read from the file is stored as well. You
      can access both with these attributes:
      
      .. attribute:: AlignmentWithSequenceReversal.read_as_aligned
      
         A :class:`SequenceWithQualities` object. The read as it was found in the file.
         
      .. attribute:: AlignmentWithSequenceReversal.read_as_sequenced
      
         A :class:`SequenceWithQualities` object. The read as it was sequenced, 
         i.e., an alias for :attr:`Alignment.read`.
      


Format-specific Alignment classes
=================================

Note: All format-specific Alignment classes take a string as argument for their constructor. This
is a line from the alignment file describing the alignment and is passed in by the corresponding
``Reader`` object. As you do not create ``Alignment`` objects yourself but get them from the ``Reader``
object you typically never call the constructor yourself.

.. class:: BowtieAlignment( bowtie_line )

   ``BowtieAlignment`` objects contain all the attributes from :class:`Alignment` and 
   :class:`AlignmentWithSequenceReversal`, and, in addition, these:
   
   .. attribute:: BowtieAlignment.reserved
   
      A string. The ``reserved`` field from the Bowtie output file. See the Bowtie manual for its meaning.

   .. attribute:: BowtieAlignment.substitutions
   
      A string. The substitutions string that describes mismatches in the format ``22:A>C, 25:C>T``
      to indicate a change from A to C in position 22 and from C to T in position 25.
      No further parsing for this is offered yet.
      
.. class:: SAM_Alignment( line )

   ``SAM_Alignment`` objects contain all the attributes from :class:`Alignment` and 
   :class:`AlignmentWithSequenceReversal`, and, in addition, these:
   
   .. attribute:: SAM_Alignment.aQual
   
      An int. The alignment quality score in Phread style encoding.

   .. attribute:: SAM_Alignment.cigar
   
      A list of :class:`CigarOperation` objects, as parsed from the extended CIGAR string. See
      :class:`CigarOperation` for details.
   
   .. attribute:: SAM_Alignment.not_primary_alignment
   
      A boolean. Whether the alignment is secondary. (See SAM format reference, flag 0x0100. See also supplementary alignments, flag 0x0800.)

   .. attribute:: SAM_Alignment.failed_platform_qc
   
      A boolean. Whether the read failed a platform quality check. (See SAM format reference, flag 0x0200.)

   .. attribute:: SAM_Alignment.pcr_or_optical_duplicate
   
      A boolean. Whether the read is a PCR or optical duplicate. (See SAM format reference, flag 0x0400.)

   .. attribute:: SAM_Alignment.supplementary
   
      A boolean. Whether the alignment is supplementary. (See SAM format reference, flag 0x0800.)

   These methods access the optional fields:

   .. attribute:: SAM_Alignment.optional_field( tag )
   
      Returns the optional field ``tag``. See SAM format reference for the defined tags (which
      are two-letter strings).
      
   .. attribute:: SAM_Alignment.optional_fields( )
   
      Returns a dict with all optional fields, using their tags as keys.
      
      

   This method is useful to write out a SAM file:

   .. method:: SAM_Alignment.get_sam_line( )
   
      Constructs a SAM line to describe the alignment, which is returned as a string.


   **Paired-end support**
   
      SAM_Alignment objects can represent paired-end data. If :attr:`Alignment.paired_end` is True,
      the following fields may be used:
      
      .. attribute:: SAM_Alignment.mate_aligned
      
         A boolean. Whether the mate was aligned
         
      .. attribute:: SAM_Alignment.pe_which
      
         A string. Takes one of the values "first", "second", "unknown" and "not_paired_end", to indicate
         whether the read stems from the first or second pass of the paired-end sequencing.
         
      .. attribute:: SAM_Alignment.proper_pair
      
         Boolean. Whether the mates form a proper pair. (See SAM format reference, flag 0x0002.)
         
      .. attribute:: SAM_Alignment.mate_start
      
         A :class:`GenomicPosition` object. The start (i.e., left-most position) of the mate's alignment.
         Note that mate_start.strand is opposite to iv.strand for proper pairs.
         
      .. attribute:: SAM_Alignment.inferred_insert_size
      
         An int. The inferred size of the insert between the reads.
         
.. function:: pair_SAM_alignments( alnmt_seq )

   This function takes a generator of :class:`SAM_Alignment` objects (e.g., 
   a :class:`BAM_Reader` object) and yields a sequence of pairs of alignments.
   A typical use may be::
   
      for first, second in HTSeq.BAM_Reader( "some_paired_end_data.sam" ):
          print("Pair, consisting of")
          print("   ", first)
          print("   ", second)
          
   Here, ``first`` and ``second`` are :class:`SAM_Alignment` objects, representing two reads
   of the same cluster. For this to work, the SAM file has to be arranged such that
   paired reads are always in adjacent lines. As the SAM format requires that the query names
   (first column of the SAM file) is the same for mate pairs, this arrangement can easily be
   achieved by sorting the SAM file lines lexicographically. 

   Special care is taken to properly pair up multiple alignment lines for the same read.

   In the SAM format, alignments for paired-end reads must be reported in paired alignment
   records. If the mate of an alignment record is missing, this fact is counted and, at the
   end, a warning stating the number of such violating reads is issued. The singleton
   alignments are yielded as pairs, with the alignment in the first or second
   element of the pair (depending on the sequencing pass it originates from) and the other 
   element is set to ``None``. 
   
.. function:: pair_SAM_alignments_with_buffer( alignments, max_buffer_size=3000000 )

   This function pairs up reads in a SAM file, in the same manner as :func:`pair_SAM_alignments`
   but does not require that mated alignments appear in adjacent records, i.e., the SAM
   file does not need to be sorted by read name beforehand. Rather, once the first alignment
   of a pair is encountered, it is stored in a buffer until its mated alignment is encountered,
   and then both are yielded together as pair. It is recommended that the data should be
   sorted by position, because then, mated alignments will typicalle not be too distant from
   each other in the file and hence only a limited number of alignments have to be held
   concurrently in the buffer, thereby reducing memory needs. To avoid overflowing the
   system's memory, the function stops and raises an exception once the number of
   alignment records held in the buffer exceeds ``max_buffer_size``.
      
.. class:: SolexaExportAlignment( line )

   ``SolexaExportAlignment`` objects contain all the attributes from :class:`Alignment` and 
   :class:`AlignmentWithSequenceReversal`, and, in addition, these:
   
   .. attribute:: SolexaExportAlignment.passed_filter
   
      A boolean. Whether the read passed the chastity filter. If ``passed_filter==False``, then
      ``aligned==False``.

   .. attribute:: SolexaExportAlignment.nomatch_code
   
      A string. For ``aligned==False``, a code indicating why no match could be found. See the 
      description of the 11th column of the Solexa Export format in the SolexaPipeline
      manual for the meaning of the codes. For ``aligned==True``, ``nomatch_code==None``.
         


Multiple alignments
===================

.. function:: bundle_multiple_alignments( sequence_of_alignments )

Some alignment programs, e.g., Bowtie, can output multiple alignments,
i.e., the same read is reported consecutively with different alignments.
This function takes an iterator over alignments (as provided by one of the 
alignment Reader classes) and bundles consecutive alignments regarding the 
same read to a list of Alignment objects and returns an iterator over these.


CIGAR strings
=============

When reading in SAM files, the CIGAR string is parsed and stored as a list of
``CigarOperation`` objects. For example, assume, a 36 bp read has been aligned
to the '+' strand of chromosome 'chr3', extending to the right from position
1000, with the CIGAR string ``"20M6I10M"``. The function :function:parse_cigar
spells out what this means:

.. doctest::

   >>> HTSeq.parse_cigar( "20M6I10M", 1000, "chr2", "+" )  #doctest:+NORMALIZE_WHITESPACE
   [< CigarOperation: 20 base(s) matched on ref iv chr2:[1000,1020)/+, query iv [0,20) >,
    < CigarOperation: 6 base(s) inserted on ref iv chr2:[1020,1020)/+, query iv [20,26) >,
    < CigarOperation: 10 base(s) matched on ref iv chr2:[1020,1030)/+, query iv [26,36) >]

We can see that the map includes an insert. Hence, the affected coordinates run from 1000
to 1030 on the reference (i.e., the chromosome) but only from 0 to 36 on the query (i.e., the read).

We can convenient access to the parsed data by looking at the attributes of the three ``CigarOperation``
objects in the list.

.. class:: CigarOperation(type, size, rfrom, rto, qfrom, qto, chrom, strand, check=True)

   This class represents a CIGAR atomic operation (e.g. 5I - insertion of 5 bases). All constructor parameters except the last one are positional, i.e. the order matters and you cannot assign them by name:

   type (str): the type of operation, must be one of M/I/D/N/S/H/P/=/X.

   size (int): the length of the operation, must be a positive integer.

   rfrom (int): the beginning of the operation in the reference genome.

   rto (int): the end of the operation in the reference genome.

   qfrom (int): the beginning of the operation in the query (sequencing read).

   qto (int): the end of the operation in the query (sequencing read).

   chrom (str): the name of the chromosome.

   strand (str): the strandness of the operation, must be +/-/. .

   check (bool, default True): whether to check the operation for internal consistency.

   Instances of this class have only one method:

   .. method:: CigarOperation.check()

      Check whether the operation is internally consistent. Returns a boolean.
  
   Instances of this class contain the following attributes:

   .. attribute:: CigarOperation.type
   
      The type of the operation. One of the letters M, I, D, N, S, H, or P. Use
      the dict **cigar_operation_names** to transform this to names:
      
      .. doctest::
      
          >>> sorted(HTSeq.cigar_operation_names.items())    #doctest:+NORMALIZE_WHITESPACE
          [('=', 'sequence-matched'),
           ('D', 'deleted'),
           ('H', 'hard-clipped'),
           ('I', 'inserted'),
           ('M', 'matched'),
           ('N', 'skipped'),
           ('P', 'padded'),
           ('S', 'soft-clipped'),
           ('X', 'sequence-mismatched')]
           
   .. attribute:: CigarOperation.size
   
      The number of affected bases, an int.
      
   .. attribute:: CigarOperation.ref_iv
   
      A :class:`GenomicInterval` specifying the affected bases on the reference. In case
      of an insertion, this is a zero-length interval.
      
   .. attribute:: CigarOperation.query_from
                  CigarOperation.query_to
                  
      Two ints, specifying the affected bases on the query (the read). In case of a
      deletion, ``query_from == query_to``. 
