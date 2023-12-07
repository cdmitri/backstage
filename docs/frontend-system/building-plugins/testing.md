---
id: testing
title: Frontend System Testing Plugins
sidebar_label: Testing
# prettier-ignore
description: Testing plugins in the frontend system
---

> **NOTE: The new frontend system is in a highly experimental phase**

# Testing Frontend Plugins

> NOTE: The new frontend system is in alpha, and some plugins do not yet fully implement it.

Utilities for testing frontend features and components are available in `@backstage/frontend-test-utils`.

## Testing extensions and components

To facilitate testing of frontend features and components, the `@backstage/frontend-test-utils` package provides a tester class which starts up an entire frontend harness, complete with a number of default features. You can then provide overrides for extensions whose behavior you need to adjust for the test run.

A number of features (frontend extensions and overrides) are also accepted by the tester. Here are some examples of how these facilities can be useful:

### Testing a single extension

In order to test an extension in isolation, you simply need to pass it into the tester factory, then call the render method on the returned instance:

```tsx
import { createExtensionTester } from '@backstage/frontend-test-utils';
import { indexPageExtension } from './plugin';

describe('Index page', () => {
  it('should render a the index page', () => {
    const tester = createExtensionTester(indexPageExtension);

    tester.render();

    expect(screen.getByText('Index Page')).toBeInTheDocument();
  });
});
```

### Testing an extension preset

There are some extensions that expect other extensions to exist, such as a page that links to another page. In that case, you can add more than one extension to the preset of features you want to render in the test, as shown below:

```tsx
import { createExtensionTester } from '@backstage/frontend-test-utils';
import { indexPageExtension, detailsPageExtension } from './plugin';

describe('Index page', () => {
  it('should link to the details page', () => {
    const tester = createExtensionTester(indexPageExtension);

    // Adding more extensions to the preset being tested
    tester.add(detailsPageExtension);
    tester.render();

    expect(screen.getByText('Index Page')).toBeInTheDocument();

    userEvent.click(screen.getByRole('link', { name: 'See details' }));

    expect(screen.getByText('Details Page')).toBeInTheDocument();
  });
});
```

### Setting config and mocking apis

If your extension requires configuration or requires implementation of APIs that aren't wired up by default, you'll have to add overrides to the preset of features being tested:

```tsx
import { createExtensionOverrides } from '@backestage/frontend-plugin-api';
import { createExtensionTester, MockConfigApi } from '@backstage/frontend-test-utils';
import {
  createApiFactory,
  configApiRef,
  analyticsApiRef,
} from '@backstage/core-plugin-api';
import { indexPageExtension } from './plugin';

describe('Index page', () => {
  it('should accepts a custom title via config', () => {
    // Mocking the config api implementation
    const configApiOverride = createApiExtension({
      factory: createApiFactory({
        api: configApiRef,
        factory: () => new MockConfigApi({
          app: {
            extensions: [{
              'page:plugin/index': {
                config : { title: 'Custom title' }
              }
            }]
          }
        }),
      }),
    });

    const tester = createExtensionTester(indexPageExtension);

    // Overriding the config api extension
    tester.add(createExtensionOverrides({
      extensions: [configApiOverride]
    });

    tester.render();

    expect(screen.getByText('Custom title')).toBeInTheDocument();
  });

  it('should capture click events in anylitics', () => {
    // Mocking the analytics api implementation
    const analyticsApiMock = new MockAnalyticsApi()

    const analyticsApiOverride = createApiExtension({
      factory: createApiFactory({
        api: analyticsApiRef,
        factory: () => analyticsApiMock,
      }),
    });

    const tester = createExtensionTester(indexPageExtension);

    // Overriding the analytics api extension
    tester.add(createExtensionOverrides({
      extensions: [analyticsApiOverride]
    }));

    tester.render();

    userEvent.click(screen.getByRole('link', { name: 'See details' }));

    expect(analyticsApiMock.getEvents()[0]).toMatchObject({
      action: 'click',
      subject: 'See details',
    });

    expect(screen.getByText('test')).toBeInTheDocument();
  });
});
```

### Rendering components in a frontend app

A component can be used for more than one extension, and it should be tested independently of the extension environment. Use the `renderInTestApp` helper to render a given component inside a Backstage test app:

```tsx
import {
  renderInTestApp,
  TestApiProvider,
} from '@backstage/frontend-test-utils';
import { stringifyEntityRef } from '@backstage/catalog-model';
import { CatalogApi, catalogApiRef } from '@backstage/plugin-catalog-react';
import { EntityDetailsComponent } from './plugin';

describe('Entity details component', () => {
  it('should render the entity name and owner', () => {
    const catalogApiMock: Partial<CatalogApi> = {
      getEntityFacets: () =>
        Promise.resolve({
          facets: {
            'relations.ownedBy': [{ count: 1, value: 'group:default/tools' }],
          },
        }),
    };

    const entityRef = stringifyEntityRef({
      kind: 'Component',
      namespace: 'default',
      name: 'test',
    });

    renderInTestApp(
      <TestApiProvider apis={[[catalogApiRef, catalogApiMock]]}>
        <EntityDetails entitRef={entityRef} />
      </TestApiProvider>,
    );

    await waitFor(() =>
      screen.getByText('The entity "test" is owned by "tools"'),
    );
  });
});
```

It's important to note that mocking APIs for components is different from mocking them for extensions. In the snippet above, we wrapped the component within a `TestApiProvider` for mocking the Catalog API.

## Missing something?

If there's anything else you think needs to be covered in the docs or that you think isn't covered by the test utilities, please create an issue in the Backstage repository. You are always welcome to contribute as well!
