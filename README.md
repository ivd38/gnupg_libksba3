GNUPG libksba 1.6.1 overflow

<pre>
gpg_error_t
_ksba_cms_parse_signed_data_part_1 (ksba_cms_t cms)
{
...
[1] err = _ksba_ber_read_tl (cms->reader, &ti);
  if (err)
    return err;
  if ( !(ti.class == CLASS_UNIVERSAL && ti.tag == TYPE_SET
         && ti.is_constructed) )
    return gpg_error (GPG_ERR_INV_CMS_OBJ);  /* not the expected SET tag */
  if (!signed_data_ndef)
    {
      if (signed_data_len < ti.nhdr)
        return gpg_error (GPG_ERR_BAD_BER); /* triplet header larger that sequence */
      signed_data_len -= ti.nhdr;
      if (!ti.ndef && signed_data_len < ti.length)
        return gpg_error (GPG_ERR_BAD_BER); /* triplet larger that sequence */
      signed_data_len -= ti.length;
    }
  algo_set_len = ti.length;
  algo_set_ndef = ti.ndef;

  if (algo_set_ndef)
    return gpg_error (GPG_ERR_UNSUPPORTED_ENCODING);
[2]  buffer = xtrymalloc (algo_set_len + 1);
     if (!buffer)
        return gpg_error (GPG_ERR_ENOMEM);

[3]  if (read_buffer (cms->reader, buffer, algo_set_len))
    {
      xfree (buffer);
      err = ksba_reader_error (cms->reader);
      return err? err: gpg_error (GPG_ERR_GENERAL);
    }

</pre>

We can set algo_set_len to arbitrary value.

If we set to (unsigned long)-1, on line #2 we have a call xtrymalloc(0).

The issue is that if application sets malloc_hooks (via ksba_set_malloc_hooks),
and set malloc function to a some function which does not accept null size - 
we can't trigger the bug. 

In any other case - we can trigger heap overflow.

In case of gpg, if sets malloc hooks and calls gcry_malloc(0) on line #2, as a result code on line #2 fails.



How to test:
<pre>
1. build libksba with asan
2. edit tests/t-cms-parser.c to open 1.cms file
3. run tests/t-cms-parser
</pre>

Asan log attached.
