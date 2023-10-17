#!/usr/bin/env python3

import sys
if sys.version_info.major < 3:
    sys.stderr.write("This script requires python3\n")
    sys.exit(1)
import os
import logging
import optparse
import webbrowser
import signal
import markdown
import pyinotify
from PyQt5.Qt import *
from PyQt5 import QtCore
from PyQt5.QtWebKitWidgets import QWebPage
from PyQt5.QtWebKitWidgets import QWebView
import requests
import xml.etree.ElementTree as ET

app_name = "mdpreview"
app_version = "2.3"

# Setup logging
level = logging.WARNING
log = logging.getLogger(app_name)
log.setLevel(level)
ch = logging.StreamHandler()
ch.setLevel(logging.DEBUG)
formatter = logging.Formatter('%(asctime)s %(levelname)8s %(name)s | %(message)s')
ch.setFormatter(formatter)
log.addHandler(ch)

extensions = (
    '.md',
    '.txt',
    '.markdown',
)

theme_items = (
    ('Bitbucket', 'bitbucket'),
    ('Dark', 'dark'),
    ('Github', 'github'),
    ('Solarized', 'solarized'),
    ('White on black', 'whiteonblack'),
)

toolbar_items = [
    {
        "id": "tb-back",
        "type": "button",
        "event": "back",
        "icon": "go-previous",
        "label": "Go back to the previous page",
        "shortcut": "B",
    },
    {
        "id": "tb-close",
        "type": "button",
        "event": "close",
        "icon": "application-exit",
        "label": "Exit mdpreview",
        "shortcut": "Escape",
    },
    {
        "id": "tb-keepontop",
        "type": "check",
        "event": "toggle_keepontop",
        "icon": "go-top",
        "label": "Keep window on top",
        "shortcut": "T",
    },
    {
        "id": "tb-savehtml",
        "type": "button",
        "event": "savehtml",
        "icon": "document-save-as",
        "label": "Save as HTML",
        "shortcut": "S",
    },
    {
        "id": "tb-edit",
        "type": "button",
        "event": "edit",
        "icon": "text-editor",
        "label": "Edit",
        "shortcut": "E",
    },
    {
        "id": "tb-theme",
        # The theme dropdown is constructed in code from theme_items. See
        # GUI._make_toolbar
    }
]

keypress_items = [
    {"key": Qt.Key_D, "event": "dump"},
    {"key": Qt.Key_G, "event": "scroll_top"},
    {"key": Qt.Key_G, "event": "scroll_bottom", "modifiers": Qt.ShiftModifier},
    {"key": Qt.Key_J, "event": "scroll_down"},
    {"key": Qt.Key_K, "event": "scroll_up"},
    {"key": Qt.Key_Q, "event": "close"},
    {"key": Qt.Key_R, "event": "reload"},
    {"key": Qt.Key_Space, "event": "scroll_pgdown"},
    {"key": Qt.Key_Backspace, "event": "back"},
    {"key": Qt.Key_Left, "event": "back"},
]

# Urlize markdown extensions taken from https://github.com/r0wb0t/markdown-urlize
# Copyright: 2-clause BSD license.
# Covers all Urlize* classes.

# Global Vars
URLIZE_RE = '(%s)' % '|'.join([
    r'<(?:f|ht)tps?://[^>]*>',
    r'\b(?:f|ht)tps?://[^)<>\s]+[^.,)<>\s]',
    r'\bwww\.[^)<>\s]+[^.,)<>\s]',
    r'[^(<\s]+\.(?:com|net|org)\b',
])

class UrlizePattern(markdown.inlinepatterns.Pattern):
    """ Return a link Element given an autolink (`http://example/com`). """
    def handleMatch(self, m):
        url = m.group(2)

        if url.startswith('<'):
            url = url[1:-1]

        text = url

        if not url.split('://')[0] in ('http','https','ftp'):
            if '@' in url and not '/' in url:
                url = 'mailto:' + url
            else:
                url = 'http://' + url

        el = ET.Element("a")
        el.set('href', url)
        el.text = markdown.util.AtomicString(text)
        return el


class UrlizeExtension(markdown.Extension):
    """ Urlize Extension for Python-Markdown. """

    def extendMarkdown(self, md):
        """ Replace autolink with UrlizePattern """
        md.inlinePatterns.register(UrlizePattern(URLIZE_RE, md),'autolink', 10)


class MDRenderer:
    """
    Render Markdown to HTML according to a theme.
    """
    def __init__(self, theme_dir, default_theme="github"):
        self.theme_dir = theme_dir
        self.default_theme = default_theme

    def get_theme_path(self, theme):
        """
        Return the full path to the theme `theme`. This is either a relative or
        absolute path if `theme` points to an existing file, or the path to oen
        of the built-in themes.
        """
        if os.path.isfile(theme):
            # Theme points to an existing file
            theme_path = theme
        else:
            theme_fname = theme + '.html'
            theme_path = os.path.join(self.theme_dir, theme_fname)
        if not os.path.isfile(theme_path):
            raise IOError("Not a valid theme file: '{}'".format(theme_path))
        return theme_path

    def render_path(self, path, theme=None):
        """
        Render the contents of local file `path` according to `theme` (or the
        default theme). If you need to render a URI ('file://' or
        'http(s);//'), use MDRenderer.render_url() instead.
        """
        log.info("Rendering path '{}' with theme '{}'".format(path, theme))
        with open(path, 'r') as f:
            md_contents = f.read()
        return(self._render_md(md_contents, theme))

    def render_url(self, url, theme=None):
        """
        Render the contents of url `url` according to `theme` (or the default
        theme). If you need to render a local path (not 'file://' or
        'http(s);//'), use MDRenderer.render_path() instead.
        """
        log.info("Rendering url '{}'".format(url))
        theme_contents = self._read_theme(theme)
        r = requests.get(url)
        md_contents = r.text
        return(self._render_md(md_contents))

    def _render_md(self, contents, theme=None):
        theme_contents = self._read_theme(theme)
        extensions = [
            'fenced_code',
            'toc',
            'tables',
            'pymdownx.tasklist',
            'pymdownx.tilde',
            UrlizeExtension()
        ]
        md = markdown.Markdown(extensions=extensions)
        md_html = md.convert(contents)
        html = theme_contents.replace('{{{ contents }}}', md_html)

        return {
            "html": html,
            "toc": md.toc_tokens
        }

    def _read_theme(self, theme):
        """
        Load theme from disk.
        """
        if theme is None:
            theme = self.default_theme
        theme_path = self.get_theme_path(theme)
        theme_contents = open(theme_path, 'r').read()
        return theme_contents


class GUI:
    """
    Main GUI class.
    """
    def __init__(self, md_renderer, keep_on_top=False, enable_toc=True):
        self.md_renderer = md_renderer
        self.keep_on_top = keep_on_top
        self.enable_toc = enable_toc
        self.cur_uri = None  # Set by load_md()
        self.cur_scroll_pos = None
        self.history = []
        self.prev_key = None
        self.settings = QSettings("electricmonk", app_name)

        icon_path = \
            os.path.join(
                os.path.dirname(
                    os.path.realpath(sys.argv[0])
                ),
                "mdpreview_icon.png"
            )
        self.win = QMainWindow()
        self.win.setWindowIcon(QIcon(icon_path))
        geometry = self.settings.value("geometry")
        if geometry is not None:
            self.win.restoreGeometry(geometry);
        self.win.closeEvent = self._ev_close_event
        if self.keep_on_top is True:
            self.win.setWindowFlags(Qt.WindowStaysOnTopHint)
        self.win.keyPressEvent = self._ev_key_press

        self._make_toolbar()
        self._make_webview()
        self._make_watchmanager()

        self.win.show()
        self.webview.setFocus()

    def _make_toolbar(self):
        """
        Generate the toolbar.
        """
        self.toolbar = self.win.addToolBar('toolbar')

        # Addd items from toolbar_items to the toolbar
        for toolbar_item in toolbar_items:
            if toolbar_item["id"] != "tb-theme":
                action = QAction(QIcon.fromTheme(toolbar_item["icon"]),
                                 toolbar_item["label"],
                                 self.win)
                action.setData(toolbar_item["id"])
                if toolbar_item["type"] == "check":
                    action.setCheckable(True)
                if toolbar_item["id"] == "tb-keepontop" and self.keep_on_top is True:
                    action.setChecked(True)
                if toolbar_item["shortcut"] is not None:
                    action.setShortcut(toolbar_item["shortcut"])
                event_cb = getattr(self, "_ev_{}".format(toolbar_item["event"]))
                action.triggered.connect(event_cb)
                self.toolbar.addAction(action)
            else:
                # Add the theme dropdown selector
                self.toolbar.addSeparator()
                self.toolbar.addWidget(QLabel("Theme "))
                self.theme_dropdown = QComboBox(self.win)
                current_index = None
                current_theme = self.md_renderer.default_theme
                for theme_index, theme_item in enumerate(theme_items):
                    self.theme_dropdown.addItem(theme_item[0], theme_item[1])
                    if theme_item[1] == current_theme:
                        current_index = theme_index
                if current_index is not None:
                    self.theme_dropdown.setCurrentIndex(current_index)
                self.theme_dropdown.currentIndexChanged.connect(self._ev_theme_dropdown_changed)
                self.toolbar.addWidget(self.theme_dropdown)

    def _make_webview(self):
        """
        Generate the central layout with the web view.
        """
        # Table of Contents tree
        self.toc_tree = QTreeView(self.win)
        self.toc_model = QStandardItemModel()
        self.toc_model.setHorizontalHeaderLabels(['Table of Contents'])
        self.toc_tree.setModel(self.toc_model)
        self.toc_root = self.toc_model.invisibleRootItem()
        self.toc_tree.clicked.connect(self._ev_toc_tree_clicked)
        if self.enable_toc is False:
            self.toc_tree.hide()

        # Webview with the rendered Markdown
        self.webview = QWebView()
        self.webview.page().setLinkDelegationPolicy(QWebPage.DelegateAllLinks)
        self.webview.linkClicked.connect(self._ev_webview_link_clicked)
        self.webview.loadFinished.connect(self._ev_webview_load_finished)

        # Splitter with the ToC left and the webview right
        self.splitter = QSplitter(Qt.Horizontal)
        self.splitter.setSizes([0, 1])
        self.splitter.addWidget(self.toc_tree)
        self.splitter.addWidget(self.webview)

        # Main widget
        self.central = QWidget();
        self.central_layout = QVBoxLayout()
        self.central_layout.setContentsMargins(0, 0, 0, 0);
        self.central_layout.addWidget(self.splitter)
        self.central.setLayout(self.central_layout)
        self.win.setCentralWidget(self.central)

    def _make_watchmanager(self):
        """
        Set up the inotify watch manager, that can watch for changes in files.
        Watches are added when loading a markdown file through self.load_md().
        """
        self.watchmanager = pyinotify.WatchManager()
        self.notifier = pyinotify.Notifier(self.watchmanager,
                                           self._watch_notifier,
                                           timeout=10)
        self.win.timerEvent = self._timerEvent
        self.win.startTimer(200)

    def _timerEvent(self, event):
        """
        Check for inotify events. Called by self.win.timerEvent.
        """
        self.notifier.process_events()
        while self.notifier.check_events():  #loop in case more events appear while we are processing
            log.debug("_timerEvent: inotify events pending")
            self.notifier.read_events()
            self.notifier.process_events()

    def _get_toolbar_action(self, id):
        """
        Helper method to get the Action that belongs to toolbar_item[id] from
        the toolbar.
        """
        for action in self.toolbar.actions():
            if action.data() == id:
                return action

    def _get_uri_abs_path(self, uri):
        """
        Get the absolute path of `uri` if it's a local file. E.g. if the
        current dir is '/home/fboender' and uri is 'file://foo/bar', then this
        function returns '/home/fboender/foo/bar'. If the uri is
        'file:///foo/bar', it returns '/foo/bar'. If the uri isn't local
        (network), returns None.
        """
        if uri.isLocalFile():
            # File on local disk
            if uri.path().startswith("/"):
                # Absolute to root dir
                abs_path = os.path.realpath(uri.path())
            else:
                # Relative to current dir
                cur_viewed_dir = os.path.dirname(self.cur_uri.path())
                abs_path = os.path.realpath(os.path.join(cur_viewed_dir, uri.path()))

            return abs_path
        else:
            return None

    def render_md(self, uri, theme=None):
        """
        Render `uri` as Markdown to HTML according to `theme`.
        """
        assert type(uri) == QUrl
        assert uri.scheme() != ''
        log.info("Rendering '{}' (isLocalFile={}) with theme '{}'".format(uri.url(), uri.isLocalFile(), theme))

        if uri.isLocalFile():
            # File on local disk
            if uri.path().startswith("/"):
                # Absolute path
                log.debug("New file is on local disk, relative to /")
                abs_path = os.path.realpath(uri.path())
            else:
                # Relative to the file we're currently viewing
                log.debug("New file is on local disk, relative to our current doc")
                if self.cur_uri is None:
                    self.cur_uri = uri
                cur_viewed_dir = os.path.dirname(self.cur_uri.path())
                abs_path = os.path.realpath(os.path.join(cur_viewed_dir, uri.path()))

                # Modify the uri so the path is absolute.
                uri.setPath(abs_path)

            # Render
            return self.md_renderer.render_path(abs_path, theme)
        else:
            # File on the intertubes
            log.debug("New file is on the network")
            return self.md_renderer.render_url(uri.url(), theme)

    def load_md(self, uri, theme=None):
        """
        Render `uri` as Markdown to HTML according to `theme` and display it in
        the web view. Also sets up the inotify events to watch for changes to
        `path`, sets the window title, etc.
        """
        assert type(uri) == QUrl
        assert uri.scheme() != ''
        log.info("Loading '{}' (isLocalFile={}) with theme '{}'".format(uri.url(), uri.isLocalFile(), theme))
        log.debug("cur uri: {}".format(self.cur_uri))
        log.debug("new uri: {}".format(uri))
        log.debug("new uri.url: {}".format(uri.url()))
        log.debug("new uri.scheme: {}".format(uri.scheme()))
        log.debug("new uri.path: {}".format(uri.path()))
        log.debug("new uri.isLocalFile: {}".format(uri.isLocalFile()))

        # Clear web view cache so we don't view cached versions.
        QWebSettings.clearMemoryCaches();

        # Stop watching the previous file
        log.info("Removing current inotify watches")
        self.watchmanager.rm_watch(list(self.watchmanager.watches.keys()))

        # Render the contents of the file or uri to HTML
        md_contents = self.render_md(uri, theme=theme)

        if uri.isLocalFile():
            # Watch the file for modifications
            abs_path = self._get_uri_abs_path(uri)
            log.info("Adding {} to the inotify watch list".format(abs_path))
            mask = pyinotify.IN_MODIFY
            self.watchmanager.add_watch(os.path.dirname(abs_path), mask)

        rel_url = uri.url()
        log.info("Setting webview contents to url {}".format(rel_url))
        self.webview.setHtml(md_contents["html"], QUrl(rel_url))
        self.set_toc(md_contents["toc"])

        # Set window title
        win_title = uri.url()
        self.win.setWindowTitle("{}: {}".format(app_name, win_title))

        # Track some metadata
        self.cur_uri = uri
        self.cur_theme = theme

        # Gray out the 'Back' button
        action = self._get_toolbar_action('tb-back')
        if len(self.history) == 0:
            action.setEnabled(False)
        else:
            action.setEnabled(True)

    def set_toc(self, toc):
        """
        Populate the Table of Contents sidebar from `toc`. `toc` is a markdown
        'toc' extension output (`toc_tokens`).
        """
        self.toc_model.removeRows(0, self.toc_model.rowCount())

        stack = []

        # Put the top-level headings on the stack
        for heading in toc:
            stack.append(
                (
                    heading["id"],
                    heading["name"],
                    self.toc_root,
                    heading["children"],
                )
            )

        # Put items from the stack in the tree. If we encouter children, push
        # them on the stack with a reference to their parent QStandardItem in
        # the tree.
        while len(stack) > 0:
            heading = stack.pop(0)

            h_id = heading[0]
            h_name = heading[1]
            h_leaf = heading[2]
            h_children = heading[3]

            h_item = QStandardItem(h_name)
            h_item.setData(h_id)
            h_item.setEditable(False)
            h_leaf.appendRow(h_item)

            for children in h_children:
                stack.append(
                    (
                        children["id"],
                        children["name"],
                        h_item,
                        children["children"],
                    )
                )

        self.toc_tree.expandAll()

    def reload(self):
        """
        Reload the currently viewed Markdown file (self.cur_uri). Stores the
        current scroll position so that when the webview `loadFinished` event
        triggers, the page is scrolled back to the previous position.
        """
        frame = self.webview.page().mainFrame()
        scroll_pos = frame.scrollPosition()

        if scroll_pos.y() == frame.scrollBarMaximum(Qt.Vertical):
            self.cur_scroll_pos = [scroll_pos.x(), -1]  # At bottom
        else:
            self.cur_scroll_pos = self._get_webview_scrollpos()

        log.info("Reloading '{}'. theme={}, scroll_pos={}".format(self.cur_uri, self.cur_theme, self.cur_scroll_pos))
        self.load_md(self.cur_uri, self.cur_theme)

    def _get_webview_scrollpos(self):
        """
        Return the current scroll position x and y coordinates of the webview.
        """
        frame = self.webview.page().mainFrame()
        scroll_pos = frame.scrollPosition()
        return  (scroll_pos.x(), scroll_pos.y())

    def _watch_notifier(self, event):
        """
        Called when changes are detected by inotify to the file we're currently
        watching. This is set up by `self.load_md()`.
        """
        log.info("inotify event: {}".format(event))
        self.reload()

    def _ev_key_press(self, keypress):
        """
        Handle key press events. Looks in `keypress_items` for actions.
        """
        for keypress_item in keypress_items:
            if (
                keypress.key() == keypress_item["key"] and
                (
                    "modifiers" not in keypress_item or
                    (int(keypress.modifiers()) & keypress_item["modifiers"] == keypress_item["modifiers"])
                )
               ):
                event_cb = getattr(self, "_ev_{}".format(keypress_item["event"]))
                event_cb()

    def _ev_back(self):
        """
        Toolbar: Go back in history.
        """
        if len(self.history) > 0:
            log.info("history: {}".format(self.history))
            uri = self.history.pop()
            log.info("Popped '{}' from the history".format(uri))
            self.load_md(QUrl(uri), self.cur_theme)

    def _ev_close_event(self, event):
        """
        Catch the user pressing the [X] button and cleanly close the app.
        """
        self._ev_close()

    def _ev_close(self):
        """
        Toolbar: Quit application
        """
        log.debug("Saving geometry")
        self.settings.setValue("geometry", self.win.saveGeometry())
        self.settings.sync()
        log.info("Quiting application")
        QCoreApplication.quit()

    def _ev_toggle_keepontop(self):
        """
        Toolbar: Toogle the Keep-on-top function
        """
        action = self._get_toolbar_action('tb-keepontop')
        if self.keep_on_top is True:
            self.keep_on_top = False
            action.setChecked(False)
            self.win.setWindowFlag(Qt.WindowStaysOnTopHint, False)
            self.win.show()
        else:
            self.keep_on_top = True
            action.setChecked(True)
            self.win.setWindowFlag(Qt.WindowStaysOnTopHint, True)
            self.win.show()

    def _ev_toggle_toc(self):
        pass

    def _ev_savehtml(self):
        """
        Toolbar: Save current Markdown file as HTML.
        """
        file_info = QFileDialog.getSaveFileName(self.win, "Save as HTML...")
        if file_info[0] != '':
            md_contents = self.render_md(self.cur_uri, self.cur_theme)
            with open(file_info[0], 'w') as f:
                f.write(md_contents["html"])

    def _ev_edit(self):
        """
        Toolbar: Edit current file.
        """
        if self.cur_uri.isLocalFile() is True:
            editor = os.environ.get("MDPREVIEW_EDITOR", "gvim")
            cmd = "{} {}".format(editor, self.cur_uri.path())
            os.system(cmd)

    def _ev_theme_dropdown_changed(self, new_index):
        """
        Toolbar: Change the theme and reload current file.
        """
        theme = theme_items[new_index]
        self.cur_theme = theme[1]
        self.reload()

    def _ev_webview_link_clicked(self, url):
        """
        Called when the user clicks a link in the current file. If it's a local
        link, open it in mdpreview. Otherwise open it in the user's browser.
        """
        log.info("Link clicked: {}".format(url.url()))
        ext = os.path.splitext(url.path())
        if url.url().startswith("#"):
            log.info("Jumping to anchor {}".format(url.url()))
            frame = self.webview.page().currentFrame()
            frame.scrollToAnchor(url.url()[1:])
        elif ext[1].lower() in extensions:
            log.info("Adding '{}' to the history".format(self.cur_uri))
            log.info("Opening '{}' in mdpreview".format(url.url()))
            self.history.append(self.cur_uri)
            self.load_md(url)
        else:
            log.info("Opening '{}' in external browser".format(url.url()))
            webbrowser.open_new_tab(url.url())

    def _ev_webview_load_finished(self, ok):
        """
        Called when the current file is done loading in the web view. Scrolls
        back to the previous position in the file we're viewing, if set. We
        need to do this in an event, otherwise the part of the document we're
        trying to scroll too may not have been loaded yet.
        """
        frame = self.webview.page().mainFrame()
        if self.cur_scroll_pos is not None:
            if self.cur_scroll_pos[1] == -1:
                x = self.cur_scroll_pos[0]
                y = frame.scrollBarMaximum(Qt.Vertical)
                log.info("Scrolling to bottom")
            else:
                x = self.cur_scroll_pos[0]
                y = self.cur_scroll_pos[1]
                log.info("Scrolling back to {}, {}".format(x, y))
            frame.setScrollPosition(QPoint(x, y))

            # Reset scroll pos, in case we load a file outside of the reload
            # mechanism (e.g. clicking a link)
            self.cur_scroll_pos = None

    def _ev_toc_tree_clicked(self, model_index):
        anchor = model_index.data(QtCore.Qt.UserRole + 1)
        frame = self.webview.page().currentFrame()
        frame.scrollToAnchor(anchor)

    def _ev_scroll_down(self):
        """
        Scroll the webview down.
        """
        frame = self.webview.page().mainFrame()
        frame.scroll(0, 60)

    def _ev_scroll_up(self):
        """
        Scroll the webview up.
        """
        frame = self.webview.page().mainFrame()
        frame.scroll(0, -60)

    def _ev_scroll_pgdown(self):
        """
        Scroll the webview down one page.
        """
        frame = self.webview.page().mainFrame()
        frame.scroll(0, 240)

    def _ev_scroll_top(self):
        """
        Scroll to top when 'GG' is pressed.
        """
        if self.prev_key == "g":
            frame = self.webview.page().mainFrame()
            frame.setScrollPosition(QPoint(0, 0))
            self.prev_key = None
        else:
            self.prev_key = "g"

    def _ev_scroll_bottom(self):
        frame = self.webview.page().mainFrame()
        x = 0
        y = frame.scrollBarMaximum(Qt.Vertical)
        frame.setScrollPosition(QPoint(x, y))

    def _ev_reload(self):
        self.reload()

    def _ev_dump(self):
        """
        Dump some debugging output to the console.
        """
        log.debug("history: {}".format(self.history))
        log.debug("geometry: {}".format(self.win.geometry()))
        log.debug("saveGeometry: {}".format(self.win.saveGeometry()))


if __name__ == "__main__":
    parser = optparse.OptionParser(version="%prog v{}".format(app_version))
    parser.set_usage(sys.argv[0] + " <file.md>")

    parser.add_option("-d",
                      "--debug",
                      dest="debug",
                      action="store_true",
                      default=False,
                      help="Show debugging info")
    parser.add_option("-k",
                      "--keep-on-top",
                      dest="keep_on_top",
                      action="store_true",
                      default=False,
                      help="Keep window on top")
    parser.add_option("-t",
                      "--theme",
                      dest="theme",
                      action="store",
                      default="github",
                      help="Theme (name or full path to .html). Default: github")
    parser.add_option("--no-toc",
                      dest="toc",
                      action="store_false",
                      default=True,
                      help="Disable the Table Of Contents sidebar")

    (options, args) = parser.parse_args()
    if len(args) < 1:
        parser.print_usage()
        sys.exit(1)

    # Enable debugging
    if options.debug:
        log.setLevel(logging.DEBUG)

    # Make sure ctrl-c on the commandline stops the application
    signal.signal(signal.SIGINT, signal.SIG_DFL)

    # Find the theme dir by trying some paths.
    bin_dir = os.path.join(os.path.dirname(os.path.realpath(sys.argv[0])))
    log.debug(bin_dir)
    theme_dirs = [
        os.path.join(bin_dir, 'themes'),
        os.path.realpath(os.path.join(bin_dir, '..', 'lib', 'mdpreview', 'themes'))
    ]
    for theme_dir in theme_dirs:
        if os.path.exists(theme_dir):
            break
    else:
        sys.stderr.write("Couldn't find a theme dir in any of {}".format(", ".join(theme_dirs)))
        sys.exit(1)

    md_renderer = MDRenderer(theme_dir=theme_dir, default_theme=options.theme)
    app = QApplication(args)
    gui = GUI(md_renderer, options.keep_on_top, options.toc)

    uri = QUrl(args[0])
    if uri.scheme() == "":
        uri.setScheme("file")
        uri.setPath(os.path.realpath(uri.path()))

    gui.load_md(uri)
    app.exec_()
