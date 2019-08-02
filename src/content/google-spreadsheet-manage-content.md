---
title: >-
  How to use Google Spreadsheet to manage content and trigger a deployment of
  your GatsbyJS site
author: jmolivas
excerpt: ''
image: sheet-gatsby.jpg
layout: post
date: '2019-03-10'
path: ''
tags:
  - Source
contentType: page
updated_at: '2019-08-02T01:30:30.735Z'
---

In early April 2019 my local meetup [Mexicali Open Source](https://mxlos.org/) was invited to talk about emerging technologies at the [Instituto Tecnológico de Mexicali](http://www.itmexicali.edu.mx/).

After happily accepting the invitation, we got together to plan and build the event site and, the tool of choice was, without much surprise, GatsbyJS. The site was pretty basic and it was used to list the speaker, date, and time of each talk. To make updating the site content easier, we decided to use a Google Spreadsheet

I will list some of the benefits of using a Google Spreadsheet

* Provides a GUI to edit content from Google Sheet.
* Multiplatform access the sheet from any device.
* Real-time collaboration by sharing the sheet with multiple people
* Backups and version control included out-of-the-box.

### How was the Gatsby site implemented?

#### The starter

For this particular case, we used the [`gatsby-v2-tutorial-starter`](https://github.com/justinformentin/gatsby-v2-tutorial-starter). I have to mention that I rather like to extend Gatsby implementations from a theme to be able to keep in sync with theme updates but also to simplify override features. But for a site this simple using a starter works great.

#### The plugin configuration

We decided to use the [gatsby-source-google-spreadsheet](https://www.gatsbyjs.org/packages/gatsby-source-google-spreadsheet/) plugin.

This plugin allows you to source all data from a Google Spreadsheet into a GraphQL type for build-time consumption.

The plugin is based on the node-sheets package and the Google Sheets API V4, which allows for better value types and column names than many of the other Gatsby Google Sheets source plugins.

Authentication shown in this example is using an api key you have created in the [google developers console](https://console.developers.google.com/).

Plugin configuration using your `gatsby-config.js` file:

```
{
  resolve: 'gatsby-source-google-spreadsheet',
  options: {
      spreadsheetId: '1BujMhjqqlo51HXFE7YyWf11579ADEv-aQRV_QerfzyE',
      worksheetTitle: 'sheet1',
      typePrefix: "GoogleSpreadsheet",
      credentials: JSON.parse(`${process.env.GOOGLE_SERVICE_ACCOUNT_CREDENTIALS}`),
  }
},

```

Note we are using a stringified version of the google-service-account data and then use the `GOOGLE_SERVICE_ACCOUNT_CREDENTIALS` environment variable in order to provide the data to the Gatsby site to avoid having the file deployed or committed by mistake on the repository.

#### Creating the ImageSharpNode from sheet data

We decided to make it simple by mapping previously uploaded speaker-avatar images with the field `avatar` stored on the sheet by adding code to `onCreateNode` at `gatsby-node.js` file.

```
if (node.internal.owner === 'gatsby-source-google-spreadsheet') {
  const avatar = (node.avatar !== null) ? node.avatar : 'avatar';
  const nodes = getNodes();
  let imageSharpNode = getImageSharp(nodes, avatar);
  if (imageSharpNode === undefined) {
    imageSharpNode = getImageSharp(nodes, 'avatar');
  }

  createParentChildLink({ parent: node, child: imageSharpNode });

  return;
}

```

Validate `node.internal.owner` is equal to `gatsby-source-google-spreadsheet` to affect only this node types.

Read avatar field value from sheet data and if null then we defaulted as `avatar`.

Next step, use the `getImageSharp` function to retrieve the `ImageSharpNode` from Gatsby and if not found, then retrieve the default `ImageSharpNode` using `avatar` as search value.

Finally, add the `childrenFile` node using the `createParentChildLink` function and passing current `node` as parent and the retrieved `imageSharpNode` as child.

#### The Index page

Add the GraphQL query to retrieve the sheet data ordered by date field.

```
export const pageQuery = graphql`
  query {
    allGoogleSpreadsheetSheet1(
     sort: { fields: [date], order: ASC }
  ) {
      edges {
        node {
          id
          name
          bio
          twitter
          website
          topic
          date
          childrenFile {
            childImageSharp {
              fluid {
                ...GatsbyImageSharpFluid
              }
            }
          }
        }
      }
    }
  }
`;

```

Note we are using the `childrenFile`node added via the `gatsby-node.js` file.

Iterating and showing the GraphQL result data on the page.

```
const Index = ({ data }) => {
  const speakers = data.allGoogleSpreadsheetSheet1.edges;
  return (
    <Layout>
      <Helmet title={'TecNerd'} />
      <Header title="TecNerd">Conociendo las nuevas tendencias tecnologicas</Header>
      <PostWrapper>
        {speakers.map(({ node }) => (
          <PostList
            key={node.id}
            cover={node.childrenFile[0].childImageSharp.fluid}
            path={node.website}
            title={node.name}
            date={node.date}
            excerpt={node.topic}
          />
        ))}
      </PostWrapper>
    </Layout>
  );
};

```

Nothing fancy here, just usage of the `map` function to iterate and pass current item properties to the `PostList` React component.

You can click here to take a look at the [PostList](https://github.com/mxlos/tecnerd/blob/master/src/components/PostList.jsx) component code.

#### Deploying the site to Netlify without leaving your Google Spreadsheet

In order to accomplish this task, it was necessary to add a script to the Google Spreadsheet. This was done by selecting the menu item `Tools` > `Script editor` and adding this code.

```
function onOpen() {
  SpreadsheetApp.getUi()
      .createMenu('Scripts')
      .addItem('Build', 'build')
      .addToUi();
}

function build() {
  UrlFetchApp.fetch('https://api.netlify.com/build_hooks/<uuid>', {
    'method': 'post',
  })
}

```

This will add a new menu item on your Google Spreadsheet menu. By clicking this you will be able to trigger a new deploy to Netlify without leaving your sheet.

![tecnerd-sheet-build.png](/sites/jmolivas/files/images/tecnerd-sheet-build.png)

Make sure you replace the `<uuid>` placeholder with the build_hooks `uuid` value from your Netlify project.

Thanks to [Richard B. Kaufman López](https://twitter.com/sparragus) for sharing with me the initial code snippet to deploy to Netlify from a Google Spreadsheet.

#### What will be great for future iterations?

* Create pages for each one of the talks, this is totally doable by adding code to `createPages` at `gatsby-node.js` file.
* Provide support to read data from diff sheet-tabs for talks, posts, social events, and others.
* Registration form using a cloud function and NoSQL storage, or maybe just sent data to a Google Spreadsheet via google form or using the API.
* Write or extend from a theme instead of using a starter.

**If interested you can:**

* Visit the site on the following link [TecNerd](https://tecnerd.netlify.com/).
* Take a read to the code at [Code repository](https://github.com/mxlos/tecnerd).

Do you think you have any other feature idea or comment you think could be interesting or useful. Let us know in the comments below.

Want to learn how to take advantage of Gatsby on your organization?.
[WeKnow](https://weknowinc.com/contact) can provide support. Feel free to [contact us](https://weknowinc.com/contact), we will be more than glad to help you.
