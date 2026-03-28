---
title: FAQ
slug: /knowledge-base/using-javelit
---

# FAQ

Here are some frequently asked questions about using Javelit. If you can't find any answer to your question here, 
please [open an issue](https://github.com/javelit/javelit/issues)or [open a discussion thread](https://github.com/javelit/javelit/discussions) in the Github project.

<Collapse title="How can I update the javelit CLI to the latest version" expanded={false} >
```bash
jbang app install --fresh --force javelit@javelit
```
</Collapse>

<Collapse title="How can I install a specific version of the javelit CLI" expanded={false} >
```bash
jbang app install --force io.javelit:javelit:${JEAMLIT_VERSION}:all@fatjar
```
</Collapse>

<Collapse title="How can I run Javelit on a subpath, behind a proxy ?" expanded={false} >
Javelit expects to be served at the root "/" of a domain. This is important for the websocket, media and page urls.
If Javelit is served behind a proxy on a sub-path, for instance `example.com/behind/proxy`:
- if the proxy sets the `X-Forwarded-Prefix` header, Javelit will work with no additional configuration
- if the header above is not set, you need to pass the `--base-path` flag manually:
   - Standalone, in Javelit CLI: `javelit run --base-path=/behind/proxy ...`.
   - Embedded, programmatically: `Server.builder(...).basePath("/behind/proxy").build()`.

If `--base-path` is set and you need to access the app on localhost directly (not behind the proxy) pass the `?ignoreBasePath=true` query parameter.  
Example: `localhost:8080/?ignoreBasePath=true`.
In dev mode, the browser automatically opens with `?ignoreBasePath=true`.

Example of Nginx configuration for running Javelit on a subpath:

```nginx
location /behind/proxy/ {
            proxy_connect_timeout 7d;
            proxy_send_timeout 7d;
            proxy_read_timeout 7d;
            proxy_http_version 1.1;
            proxy_set_header X-Forwarded-Prefix behind/proxy;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
            proxy_pass http://localhost:8080/;
   }

```
</Collapse>

<Collapse title="How can I customize the CSS? How can I use a custom theme? What about dark mode?">
Dark mode - as in switching between light and dark mode - is not supported yet.  
You can pass your own custom theme as an `.html` file through custom headers.
```bash
javelit run --headers-file custom_theme.html YourApp.java
```
Or embedded:
```
Server.builder(....).headersFile(<Path to custom_theme.html>).build().start();
```
You can find example of custom themes in this [folder](https://github.com/javelit/javelit/tree/main/examples/dark_mode).
If you wish to make an LLM generate a custom theme, provide the [original theme](https://github.com/javelit/javelit/blob/main/src/main/resources/design_system.html.mustache) 
as input.

For more info, join the discussion in this [thread](https://github.com/javelit/javelit/discussions/39#discussioncomment-14841955).
</Collapse>

{/*

Here are some frequently asked questions about using Streamlit. If you feel something important is missing that everyone needs to know, please [open an issue](https://github.com/streamlit/docs/issues) or [submit a pull request](https://github.com/streamlit/docs/pulls) and we'll be happy to review it!

- [Sanity checks](/knowledge-base/using-javelit/sanity-checks)
- [How can I make Streamlit watch for changes in other modules I'm importing in my app?](/knowledge-base/using-javelit/streamlit-watch-changes-other-modules-importing-app)
- [What browsers does Streamlit support?](/knowledge-base/using-javelit/supported-browsers)
- [Where does st.file_uploader store uploaded files and when do they get deleted?](/knowledge-base/using-javelit/where-file-uploader-store-when-deleted)
- [How do you retrieve the filename of a file uploaded with st.file_uploader?](/knowledge-base/using-javelit/retrieve-filename-uploaded)
- [How to remove "· Streamlit" from the app title?](/knowledge-base/using-javelit/remove-streamlit-app-title)
- [How to download a file in Streamlit?](/knowledge-base/using-javelit/how-download-file-streamlit)
- [How to download a Pandas DataFrame as a CSV?](/knowledge-base/using-javelit/how-download-pandas-dataframe-csv)
- [How can I make `st.pydeck_chart` use custom Mapbox styles?](/knowledge-base/using-javelit/pydeck-chart-custom-mapbox-styles)
- [How to insert elements out of order?](/knowledge-base/using-javelit/insert-elements-out-of-order)
- [How do I upgrade to the latest version of Streamlit?](/knowledge-base/using-javelit/how-upgrade-latest-version-streamlit)
- [Widget updating for every second input when using session state](/knowledge-base/using-javelit/widget-updating-session-state)
- [How do I create an anchor link?](/knowledge-base/using-javelit/create-anchor-link)
- [How do I enable camera access?](/knowledge-base/using-javelit/enable-camera)
- [Why does Streamlit restrict nested `st.columns`?](/knowledge-base/using-javelit/why-streamlit-restrict-nested-columns)
- [What is serializable session state?](/knowledge-base/using-javelit/serializable-session-state)

*/}
