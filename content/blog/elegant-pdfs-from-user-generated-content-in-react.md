---
path: react-pdf-w-user-generated-content
date: 2020-03-02T18:21:31.257Z
title: Elegant PDFs from User-Generated Content in React
description: >-
  Learn how to use DraftJS, Redraft, and React PDF to generate elegant PDFs with
  user-generated content.
---
Converting user generated HTML into a PDF has always been a pain. You’re stuck with solutions like [wkhtmltopdf](https://wkhtmltopdf.org/), [PhantomJS](https://coderwall.com/p/5vmo1g/use-phantomjs-to-create-pdfs-from-html), or [Headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome#create_a_pdf_dom). The problem I have found with all of these tools is that they are either slow, fail to match styles, or require running on a server. At CompanyCam we allow our users to generate reports, some of which are hundreds of pages long. No matter how you slice it, that makes any of the options above much more difficult, especially if you allow the user to preview their PDF.

In this post we will look at how we use a combination of React PDF, DraftJS, and Redraft to craft a PDF from user-generated content that is instantly available in the user's browser for preview and download.

Since we use several libraries I think it may be helpful give a quick overview and what each library is used for.

### React PDF
[React PDF](https://react-pdf.org/) is a PDF renderer built for React. It includes several components that represent different aspects of a PDF document such as a `Document`, `Page`, and `View`. It can output to a file on the server or a `Blob`  in the browser. It provides an interface for layout, styling, and asset fetching.

### DraftJS
[DraftJS](https://draftjs.org/) describes itself as a "Rich Text Editor Framework for React". Draft was developed by Facebook and is extremely flexible and extensible.

### Redraft
[Redraft] is a middleman of sorts and renders the output from DraftJS, allowing you to override the default elements that correspond with different *blocks*.

### React-PDF
No, this isn't a duplicate. Confusingly, it has a similar name, but it a completely different library. [React-PDF](https://github.com/wojtekmaj/react-pdf) is a library to display PDFs on the web as an `svg` or `canvas`. As a tip, this library  **doesn't** have a "@" in the name.

## The Flow
CompanyCam allows users to build reports using photos they've captured in our app. Users can add comments using a WYSIWYG editor and optionally choose if they want to display metadata such as when the photo was captured, who took the photo, and the project it belongs to.

When the user selects to generate a PDF of a report we launch a modal and finish fetching the pages of entries. Once we have the data we can get down to work. We pass the report to our `PDF` component, which is responsible for bootstrapping some libraries, generating a PDF and converting it to a blob, and then rendering the preview.

We start by passing our `Document` component to the `pdf` function imported from  `@react-pdf/renderer`. The `Document` component will be shown later, for now know that it uses `@react-pdf` components to represent entires in the report.

```
import { pdf } from ‘@react-pdf/renderer’;

Const PDF = ({ report }) => {
  const [loading, setLoading] = useState(true);
  const [documentURL, setDocumentURL] = useState();
  Const [numPages, setNumPages] = useState(0);

  useEffect(
    () => {
      const generateBlob = async () => {
        setLoading(true);
        const blob = await pdf(
          <Document report={report} imageSize={imageSize} />,
        ).toBlob();
        setLoading(false);
        setDocumentURL(window.URL.createObjectURL(blob));
      };

      if (report) {
        generateBlob();
      } else {
        setLoading(false);
        setDocumentURL(null);
      }
    },
    [report],
  );
};
```

We await the `pdf` function and then call `toBlob` to get a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) object. We generate a URL for the blob using `URL.createObjectURL` and then store that in the state. We can use the URL of the blob to render a preview of the document and to allow the user to download the file.

```
import { pdfjs, Document as PDFDocument, Page as PDFPage } from ‘react-pdf’;

pdfjs.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjs.version}/pdf.worker.js`;

const PDF = ({ report }) => {
  // PDF generation omitted here.

  if (loading) {
    return <Spinner message="Fetching Images" />;
  }

  return (
    <React.Fragment>
      {loading && <div>Rendering PDF…</div>}
      <div>
        <a
          href={documentURL}
          className="ccb-blue-small"
          style={{ margin: ‘auto 0 0’ }}
          download={filename}
        >
          Download PDF
        </a>
      </div>
      <PDFDocument
        file={documentURL}
        onLoadSuccess={(result) => setNumPages(result.numPages)}
        loading={<Spinner />}
      >
        <PDFPage
          renderMode="svg"
          pageNumber={currentPage}
        />
      </PDFDocument>
    </React.Fragment>
  );
}
```

Once we have the documentURL generated by `@react-pdf` we can render a preview using `react-pdf`. We pass the `PDFDocument` component the `documentURL` as the file. Once the document has loaded it will call `onLoadSuccess` and tell us some info such as how many pages the PDF contains. Inside of that, we pass a `PDFPage` component and tell it to either render using a `canvas` or `svg` and which page we want to display.

You will notice that we can pass the `documentURL` as the `href` and a filename to the `download` prop for the `a` tag and it will download the blob with the filename specified. This is handy because the user can instantly download the generated PDF.

I omitted the code that shows how we change the page for brevity. The code is a simple click handler that increments or decrements the `currentPage` variable making sure it stay in bounds.

At this point we know how to render and display the `Document` component, but what does `Document` *do* and how do we convert HTML to `@react-pdf` components?

## Document Rendering and Content Conversion

React-PDF handles almost everything for you out of the box so the `Document` component is fairly simple.

```
import {
  Document as PDFDocument,
  StyleSheet,
  Page,
  Font,
} from ‘@react-pdf/renderer’;
import PageFooter from ‘./PageFooter’;
import CoverPage from ‘./CoverPage’;
import Entry from ‘./Entry’;

Font.register({
  family: ‘Public Sans’,
  fonts: [
    { src: ‘https://cdn.companycam.com/fonts/PublicSans-Regular.ttf’ },
    {
      src: ‘https://cdn.companycam.com/fonts/PublicSans-SemiBold.ttf’,
      fontWeight: 700,
    },
  ],
});

// Allow users to include emojis in markup
Font.registerEmojiSource({
  format: ‘png’,
  url: ‘https://twemoji.maxcdn.com/2/72x72/‘,
});

const styles = StyleSheet.create({
  // omitted for brevity
});

const Document = ({ report }) => {
  const { 
    entries, 
    settings, 
  photoCount,
    company,
    title,
    subtitle,
    createdAt,
    featuredEntry
  } = report;

  const { name, logoLargeUrl } = company;
  const featuredPhoto = featuredEntry.assetPreviewLarge;

  return (
    <PDFDocument>
      <Page style={pageStyles} size="LETTER">
        <PageFooter title={title} />
        <CoverPage
          companyName={name}
          logo={logoLargeUrl}
          title={title}
          subtitle={subtitle}
          createdAt={createdAt}
          featuredPhoto={featuredPhoto}
          photoCount={photoCount}
        />
      </Page>
      <Page style={pageStyles} size="LETTER">
        <PageFooter title={title} />
        {entries.map((entry) => {
          return (
            <Entry
              key={entry.id}
              entry={entry}
              settings={settings}
            />
          );
        })}
      </Page>
    </PDFDocument>
  );
};

export default Document;
```

The wrapping component is a `Document` component imported from `@react-pdf`. All components inside of this component must be a `@react-pdf` component.

The `Page`, `CoverPage` and `PageFooter` are all self explanatory so I won’t show the code. We render a page at the beginning of the document that includes the company's name and title of the report. We then render an `Entry` for each item in the report. The interesting part here is that we wrap all entries in a single `Page` component. React PDF has a pretty advanced layout engine. If the content is going to overflow the page, it will wrap to a new page. You can also pass a `break` prop to add a page break before the component being rendered.

I've included the `Entry` component for completeness. The only interesting part is that it renders a `RichText` component.

```
import React from 'react';
import { StyleSheet, View, Image, Text, Link } from '@react-pdf/renderer';
import RichText from '../RichText';

const styles = StyleSheet.create({
  // ...omitted
});

const Entry = ({ entry }) => {
  const { item, pageBreak } = entry;
  const { assetPreview } = item;

  return (
    <View style={containerStyles} wrap={false} break={pageBreak}>
      <Link
        src={assetPreviewLarge}
        style={imageContainerStyles}
        target="_blank"
      >
        <Image
          style={imageStyles}
          source={{
            uri: assetPreview,
            headers: { Pragma: 'no-cache', 'Cache-Control': 'no-cache' },
          }}
        />
      </Link>
      <View style={contentStyles}>
        <RichText note={entry.notes} />
      </View>
    </View>
  );
};

export default Entry;
```

## Redraft Meet React PDF

So far, most of the code I have shown you isn’t anything earth shattering. I included it so you could have a better understanding of the hierarchy and flow. In the `RichText` component we actually do some more interesting work.

```
import {
  EditorState,
  ContentState,
  convertToRaw,
  convertFromHTML,
} from ‘draft-js’;
import redraft from ‘redraft’;

const RichText = ({ note }) => {
  const blocksFromHTML = convertFromHTML(note);
  const initialState = ContentState.createFromBlockArray(
    blocksFromHTML.contentBlocks,
    blocksFromHTML.entityMap,
  );

  const editorState = EditorState.createWithContent(initialState);
  const rawContent = convertToRaw(editorState.getCurrentContent());

  return redraft(rawContent, renderers, { blockFallback: ‘unstyled’ });
};

export default RichText;
```

The component is quite simple; it handles converting the HTML to a format that is understood by DraftJS using `convertFromHTML`. 
> If you’re using DraftJS to generate the HTML, you can store the raw JS object from Draft using `convertToRaw` while editing.

Once we have converted the HTML into _blocks_, we initialize a `ContentState` and an `EditorState`. We do all of that to finally get the `rawContent`. The raw content is a regular ole’ JS object. I recommend you log it to the console so you can see how parseable the data is. Finally we’re at the cool part, we return the result of calling `redraft` with the `rawContent`, `renderers`, and some options.

## Renderers

The `renderers` object is a object that maps from the original type to the new component. Redraft uses the `renderers` object and iterates over the blocks, inline elements, and entities in the raw JS object, calling the corresponding function based on the type. DraftJS [supports a handful of elements out of the box](https://draftjs.org/docs/advanced-topics-custom-block-render-map), so we make a compatible React PDF component for each of the types that we support. Let’s take a look at our `renderers`:

```
import { StyleSheet, View, Text, Link } from ‘@react-pdf/renderer’;

const renderers = {
  inline: {
    // The key passed here is just an index based on rendering order inside a block
    BOLD: (children, { key }) => {
      return (
        <Text key={`bold-${key}`} style={{ fontWeight: 700 }}>
          {children}
        </Text>
      );
    },
    ITALIC: (children, { key }) => {
      return (
        <Text key={`italic-${key}`} style={{ fontStyle: ‘italic’ }}>
          {children}
        </Text>
      );
    },
    UNDERLINE: (children, { key }) => {
      return (
        <Text key={`underline-${key}`} style={{ textDecoration: ‘underline’ }}>
          {children}
        </Text>
      );
    },
  },
  /**
   * Blocks receive children and depth
   * Note that children are an array of blocks with same styling,
   */
  blocks: {
    unstyled: (children, { keys }) => {
      return children.map((child, index) => {
        return (
          <View key={keys[index]}>
            <Text style={styles.text}>{child}</Text>
          </View>
        );
      });
    },
    ‘header-one’: (children, { keys }) => {
      return children.map((child, index) => {
        return <HeadingOne key={keys[index]}>{child}</HeadingOne>;
      });
    },
    ‘unordered-list-item’: (children, { depth, keys }) => {
      return (
        <UnorderedList key={keys[keys.length - 1]} depth={depth}>
          {children.map((child, index) => (
            <UnorderedListItem key={keys[index]}>{child}</UnorderedListItem>
          ))}
        </UnorderedList>
      );
    },
    ‘ordered-list-item’: (children, { depth, keys }) => {
      return (
        <OrderedList key={keys.join(‘|’)} depth={depth}>
          {children.map((child, index) => (
            <OrderedListItem key={keys[index]} index={index}>
              {child}
            </OrderedListItem>
          ))}
        </OrderedList>
      );
    },
  },
  /**
   * Entities receive children and the entity data
   */
  entities: {
    // key is the entity key value from raw
    LINK: (children, data, { key }) => (
      <Link key={key} src={data.url}>
        {children}
      </Link>
    ),
  },
};
```

As you can see we return a text element for all of the `inline` styles and apply the appropriate styles to match the type. For each of the blocks we created our own components that have applied styles to match the formatting required. For example, here is `HeaderOne`:

```
const styles = StyleSheet.create({
  headingOne: {
    marginBottom: 4,
    color: ‘#3a4b56’,
    fontWeight: 700,
    fontFamily: ‘Public Sans’,
    lineHeight: 1.35,
    fontSize: 12,
  },
});

const HeadingOne = ({ children }) => {
  return (
    <View>
      <Text style={styles.headingOne}>{children}</Text>
    </View>
  );
};
```

It a simple `Text` component wrapped in a `View`. By default a `View` will span the full width of the parent. Think of this is a block element in HTML.

One of the more interesting components is for lists. React PDF doesn’t have a built in component for ordered or unordered lists, so we made our own. Boiling it down, a list is just a block element with nested block elements prefixed with either a number or bullet before the entry.

```
const styles = StyleSheet.create({
  list: {
    marginBottom: 8,
    marginLeft: 6,
  },
  listItem: {
    marginBottom: 4,
  },
  listItemText: {
    color: ‘#6b7880’,
    fontFamily: ‘IBM Plex Serif’,
    fontSize: 10,
    lineHeight: 1.45,
  },
});

const UnorderedList = ({ children, depth }) => {
  return <View style={styles.list}>{children}</View>;
};

const UnorderedListItem = ({ children }) => {
  return (
    <View style={styles.listItem}>
      <Text style={styles.listItemText}>
        • &nbsp;<Text>{children}</Text>
      </Text>
    </View>
  );
};

const OrderedList = ({ children, depth }) => {
  return <View style={styles.list}>{children}</View>;
};

const OrderedListItem = ({ children, index }) => {
  return (
    <View style={styles.listItem}>
      <Text style={styles.listItemText}>
        {index + 1}. &nbsp;<Text>{children}</Text>
      </Text>
    </View>
  );
};
```

And there you have it. You have taken user input, parsed it using DraftJS and converted it into elements that can be styled and rendered using React PDF. Implementing new types is as easy as adding a new entry in `renderers` and styling it.

One place I would like to try using Redraft is to render React Native components from HTML instead of relying on a web view in our mobile app.

For the full code from the example you can check out [this gist](https://gist.github.com/chadwilken/caa6d945b5d5505fe1788f53af2be244).
