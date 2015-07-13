# How to use Uzbek wiktionary offline

In this tutorial I'll explain how to look up words from the
[Uzbek Wiktionary](https://uz.wiktionary.org) using the
[dict](http://linux.die.net/man/1/dict) command even when you're offline.
The resulting dictionary can be converted into multiple other dictionary
formats and used by programs such as
[stardict](https://code.google.com/p/stardict-3/),
[goldendict](http://goldendict.org/),
and [Lingvo](http://www.lingvo.com/). The tutorial doesn't cover the conversion
process. The following instructions were carried out on Arch Linux.

1. Download uzwiktionary dump from https://dumps.wikimedia.org/uzwiktionary/20150702/.
   I downloaded uzwiktionary-20150702-pages-meta-current.xml.bz2 and extracted
   it, getting uzwiktionary-20150702-pages-meta-current.xml as a result.

2. Using [wikiextractor](https://github.com/attardi/wikiextractor) convert the
   downloaded XML into HTML. This tool is great because it also expands templates.
   `WikiExtractor.py --html uzwiktionary-20150702-pages-meta-current.xml`
   This will output a file called text.txt. The text file is an XML file
   (without the xml declaration) and each doc element contains the page contents.

3. Add `<?xml version="1.0" encoding="UTF-8"?>` to the beginning of text.txt and
   save.

4. Wrap HTML content fo the XML tag so that we can retrieve it using a script:
   ```python
   newtext = []
   with open('text.txt', 'r') as f:
      for line in f.readlines():
          if line.startswith('<doc '):
              newtext.append(line)
              newtext.append('<![CDATA[')
          elif line.startswith('</doc>'):
              newtext.append(']]>')
              newtext.append(line)
          else:
              newtext.append(line)

   with open('newtext.xml', 'w') as f:
       f.write("".join(newtext))
   ```
   The above code will output a new file called newtext.xml

5. Now we can extract each page and save it as a dictionary item. Note that the
   wikipages contain headings without content, so we'll be using regex to get
   the sections we need only. We will be using [html2text]
   (https://github.com/aaronsw/html2text) to convert html to text.

   ```
   import re
   import xml.etree.ElementTree as ET
   import html2text

   tree = ET.parse('newtext.txt')
   root = tree.getroot()

   words = {}
   for doc in root.findall('doc'):
       word = doc.attrib['title']
       # extract the morphologic properties
       morf_search = re.search('<h2>Morfologik va sintaktik xususiyatlari</h2>\n(.*)\n<h2>Aytilishi</h2>', doc.text, re.M)
       morf_text = ''
       if morf_search:
           morf_text = '<b>' + morf_search.group(1) + '</b><br /><br />'
       # extract the meanings
       meanings_search = re.search('<h3>Maʼnosi</h3>\n((.|\n)*)\n<h3>Sinonimlari</h3>', doc.text, re.M)
       if meanings_search:
           result = ''
           if morf_text:
               result = morf_text
           result += meanings_search.group(1)
           # transform HTML to text
           text = html2text.html2text(result)
           # save the word and meaning. also escape newline and tab chars
           words[word] = text.replace('\n', '\\n').replace('\t', '\\t')

   with open('uzb-thesaurus-c5.txt', 'w') as f:
       converted_words = ['_____\n\n%s;%s' % (word, meaning) for word, meaning in words.items()]
       f.write("\n".join(converted_words))
   ```
   The above code will generate a new file called uzb-thesaurus-c5.txt which can be used
   with dictd.

6. Generate the dict and index files from uzb-thesaurus-c5. First we need to
   install dictd: `sudo pacman -S dictd`
   ```sh
   dictfmt -c5 --headword-separator ';' --break-headwords --utf8 \
    --allchars -s "O‘zbek tilining izohli lug‘ati" uzb-thesaurus \
    < uzb-thesaurus-c5
   ```

7. Compress the dict file
   ```sh
   dictzip uzb-thesaurus.dict
   ```

8. Copy the files to `/usr/share/dictd/`:
   ```sh
   sudo cp uzb-thesaurus.dict.dz /usr/share/dictd
   sudo cp uzb-thesaurus.index /usr/share/dictd
   ```

9. Edit the config file `/etc/dict/dictd.conf` and add the following lines:
   ```
   database uzb-thesaurus {
       data "/usr/share/dictd/uzb-thesaurus.dict.dz"
       index "/usr/share/dictd/uzb-thesaurus.index"
   }
   ```

10. Restart the dict service:
    ```sh
    sudo systemctl restart dictd
    ```

11. Use gnome dictionary, goldendict or dict command line tool to look up
    the words:
    ```sh
    dict kitob
    ```

12. The end
