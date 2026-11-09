# Third-party content in the catalog

The catalog's own files are Apache-2.0 ([LICENSE](LICENSE)); the app icons in it are other people's,
under their own terms. The apps themselves are not here: a version file names its images by
digest, and a node pulls them from their publishers.

## The catalog's app icons — Apache-2.0 collection, each mark its project's

`catalog/latest.json` carries one vector mark per listed app, inline as a data URI:

- [homarr-labs/dashboard-icons](https://github.com/homarr-labs/dashboard-icons) (Apache-2.0) —
  the collection every mark but one is taken from, unchanged but for the single-colour marks
  (Frigate, Karakeep, Vaultwarden), which are black or white with the reader's colour scheme,
  as the collection's own dark and light variants are. The marks themselves are the
  logos and trademarks of their respective projects, used to identify them.
- [Memos](https://www.usememos.com) (MIT) — its logo from the project's own repository
  (`web/public/logo.svg`), because the collection's copy is a bitmap.
