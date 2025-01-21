# Automation Framework with JavaScript, WebDriverIO, Selenium webdriver, Cucumber, Allure, NPM.

WebDriverIO can be run on the WebDriver protocol for true cross-browser testing, as well as on Chrome DevTools protocol for Chromium-based automation using Puppeteer. WebDriverIO can do the same things that Selenium can, but also comes with lots of integrations with popular test automation tools and plugins.


This was the project I worked before for about 5 years, code could not be shared, just share WDIO testrunner config file for demonstrating the automation framework.



# wdio.conf.js - This includes e2e, ZAP, Axe(accessibility) and email testing.
<pre><code>
const fs = require('fs-extra');
const path = require('path');
const nconf = require('nconf');
const _ = require('lodash');
const { addAttachment, addStep } = require('@wdio/allure-reporter').default;
require('./preload')();

const baseChromeOptionsArgs = ['--disable-extensions', '--disable-infobars'];
const headlessChromeOptionsArgs = ['--no-sandbox', '--headless=new'];
const acceptInsecureCertCapabilities = {
  acceptInsecureCerts: true,
};
const zapCapabilities = {
  proxy: {
    proxyType: 'manual',
    httpProxy: `${nconf.get('zap:hostname')}:8080`,
    sslProxy: `${nconf.get('zap:hostname')}:8080`,
    noProxy:
      '127.0.0.1,localhost,*.quantcount.com,*.quantcast.mgr.consensu.org,*.mgr.consensu.org,*.cmp.quantast.com,*.quantcast.com,*.quantserve.com,*.bugherd.com,*.googletagmanager.com,*.gstatic.com,*.googleusercontent.com,*.google.com,*.s3.us-west-2.amazonaws.com,*.gravatar.com,*.googleapis.com,
  },
};
const getTagExpression = function () {
  const partnerServerUri = nconf.get('partnerServerUri');
  switch (partnerServerUri) {
    case 'https://x1.com':
      return '@demo and not @skip';
    case 'https://y1.com':
      return '@sandbox and not @skip';
    case 'https://z1.com':
      return '@production and not @skip';
    default:
      if (nconf.get('email').enabled) {
        return '@email and not @skip';
      } else if (nconf.get('accessibility').enabled) {
        return '@accessibility and not @skip';
      } else if (nconf.get('regression').enabled) {
        return '@regression and not @skip';
      } else {
        return '@ci and not (@skip and @regression and @email and @accessibility)';
      }
  }
};

const getSpecs = function () {
  let specs = ['../specs/**/*.feature'];
  if (nconf.get('email:enabled')) {
    specs.unshift('../specs/xxxWorkflows/emails/advertiserApprovalRejectionEmail.feature');
  }
  return specs;
};

exports.config = {
  capabilities: [
    {
      browserName: 'chrome',
      'goog:chromeOptions': {
        args: nconf.get('headless') ? baseChromeOptionsArgs.concat(headlessChromeOptionsArgs) : baseChromeOptionsArgs,
      },
    },
  ],
  coloredLogs: true,
  cucumberOpts: {
    backtrace: true,
    failFast: true,
    require: ['./step-definitions/**/*.js'],
    requireModule: ['@babel/register'],
    timeout: nconf.get('cucumberTimeout:default'),
    tagExpression: getTagExpression(),
  },
  framework: 'cucumber',
  hostname: nconf.get('hostname'),
  path: nconf.get('path'),
  port: nconf.get('port'),
  logLevel: 'silent',
  maxInstances: _.toNumber(nconf.get('maxInstances')),
  reporters: [
    'spec',
    [
      'allure',
      {
        outputDir: './reports/allure-results',
        disableWebdriverStepsReporting: !nconf.get('enableWebdriverStepsReporting'),
        useCucumberStepReporter: true,
      },
    ],
  ],
  services: nconf.get('services'),
  specs: getSpecs(),
  specFileRetries: nconf.get('specRetryAttempts'),
  sync: false,
  waitforTimeout: nconf.get('webdriverIOTimeout:default'),
  waitforInterval: 1000,
  onPrepare: () => {
    fs.emptydir('./reports/');
  },
  beforeFeature: function () {
    browser.setWindowSize(1366, 1080);
  },
  beforeStep: function (_step, _scenario, context) {
    if (typeof context.publishers === 'undefined') {
      context.advertisers = [];
      context.publishers = [];
      context.sites = [];
      context.products = [];
      context.segmentGroups = [];
      context.inbox = {};
      context.lineItems = [];
      context.creatives = [];
      context.consoleErrors = [];
      context.clients = [];
      context.campaigns = [];
      context.teams = [];
      context.terms = [];
    }
  },
  afterStep: async function (step, scenario, result, context) {
    const logType = 'browser';
    const availableLogTypes = await browser.getLogTypes();

    if (_.includes(availableLogTypes, logType)) {
      const allLogs = await browser.getLogs(logType);
      let ignoredErrors = [];
      const browserLogs = allLogs.filter(({ level, message }) => {
        let isMatch;
        if (level === 'SEVERE') {
          // skip forecasting data availability checking error.
          const errorRegexPattern =
            /campaigns\/[\w-]+\/line-items\/[\w-]+\?advertiserId=\d+&parentId=[\w-]+ - Failed to load resource: the server responded with a status of 400 \(\)/;
          const errorForecastRegexPattern = /There is not enough forecasting data available yet/;
          isMatch = errorRegexPattern.test(message) || errorForecastRegexPattern.test(message);
        }
        const toFilter =
          level === 'SEVERE' &&
          !message.includes('https://x.skimresources.com/?provider=lotame') &&
          // https://github.com/reactwg/react-18/discussions/82
          !message.includes("Can't perform a React state update on an unmounted component") &&
          !isMatch;
        if (!toFilter) {
          ignoredErrors.push(message);
        }
        return toFilter;
      });
      if (!_.isEmpty(ignoredErrors)) {
        // eslint-disable-next-line no-console
        console.log('Check allure reports for ignored browser errors.');
        await browser.takeScreenshot();
        await addAttachment('ignored_browser_errors', {
          step: {
            name: `${step.keyword} ${step.text}`,
            location: scenario.uri,
          },
          errors: ignoredErrors,
        });
      }
      if (!_.isEmpty(browserLogs)) {
        const consoleErrors = {
          step: {
            name: `${step.keyword} ${step.text}`,
            location: scenario.uri,
          },
          logs: browserLogs,
        };
        await browser.takeScreenshot();
        await addAttachment('console_errors', consoleErrors);
        await addStep('Checking browser console', [], 'failed');
        context.consoleErrors.push(consoleErrors);
      }
    }
    if (result.error) {
      await browser.takeScreenshot();
      if (!_.isUndefined(result.error.response)) {
        const errorAndResponse = `Error: ${JSON.stringify(result.error, null, 2)}
Response_Data: ${JSON.stringify(_.get(result.error, 'response.data'))}`;
        await addAttachment('error_response', errorAndResponse);
      }
      const stepDetails = {
        name: `${step.keyword} ${step.text}`,
        location: scenario.uri,
      };
      await addAttachment('step_details', JSON.stringify(stepDetails, null, 2));
      let cucumberWorld = context;
      cucumberWorld.loggedInUser = _.omit(cucumberWorld.loggedInUser, ['password']);
      cucumberWorld.loggedInUser.users = _.map(cucumberWorld.loggedInUser.users, function (user) {
        return _.omit(user, ['password']);
      });
      await addAttachment('cucumber_world', JSON.stringify(cucumberWorld, null, 2));
      await addAttachment('html', await (await $('body')).getHTML());
    }
  },
  after: function () {
    const mediaAccessibilityReports = [];
    const publisherAccessibilityReports = [];
    const commonAccessibilityReports = [];

    const reportFolderPath = './reports/axe-report';
    const badgeColor = {
      warning: '#ffc107',
      success: '#28a745',
    };

    if (fs.existsSync(reportFolderPath)) {
      const files = fs.readdirSync(reportFolderPath);

      if (_.isEmpty(files)) return;

      _(files)
        .reject((file) => file.includes('index.html'))
        .forEach((file) => {
          const fileContent = fs.readFileSync(`${reportFolderPath}/${file}`, 'utf8');
          const splittedText = fileContent.split('badge-')[1].split('">');
          const badgeClass = splittedText[0];
          const violations = parseInt(splittedText[1].split('<')[0]);

          const fileLinks = `XXX`;

          switch (true) {
            case file.startsWith('media-'):
              mediaAccessibilityReports.push(fileLinks);
              break;
            case file.startsWith('publisher-'):
              publisherAccessibilityReports.push(fileLinks);
              break;
            default:
              commonAccessibilityReports.push(fileLinks);
              break;
          }
        });

      const htmlContent = `XXX`;

      const indexPath = path.join(reportFolderPath, 'index.html');
      fs.writeFileSync(indexPath, htmlContent);
    }
  },
};

if (nconf.get('acceptInsecureCerts')) {
  _.merge(exports.config.capabilities[0], acceptInsecureCertCapabilities);
}
if (nconf.get('zap:enabled')) {
  _.merge(exports.config.capabilities[0], zapCapabilities);
}
</code></pre>
