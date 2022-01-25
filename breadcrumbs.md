I will relate here (in the future github.io page) my open source programming endeavors.
They will probably be mostly raku related (rakudo, nqp and MoarVM)
with an emphasis to everything connex to the rakudo grammar engine,
especially slangs.
There will also probably be stuff about typescript, [vscode](https://en.wikipedia.org/wiki/Visual_Studio_Code)
 (particularly extensions),
[wasm](https://en.wikipedia.org/wiki/WebAssembly) (maybe in relation to rust) and sveltejs.


# Writing a rakudo parse tree browser

## Specification

The user moves the cursor on the parsed file editor and the bbar 
displays the path from the top of the parse tree to the leaf token.
On the right a view displays the rules used for the parsing.

### An access point to documentation

Eventually this should be a way to learn about the language and about 
raku grammar. When clicking a variable would open the doc about 
variables.

It will be first used to document my upcoming slangs.

Also later AST can enter into the game.



http://www.parsifalsoft.com/gloss.html
https://nearley.js.org/docs/glossary

A while ago, I dabbled in writing a parse tree browser using [Svelte](https://en.wikipedia.org/wiki/Svelte).
I had written some code to generate json from the 
[parse tree](https://en.wikipedia.org/wiki/Parse_tree) of a rakudo 
file. Now I have changed my mind and want to use a vscode extension to implement the 
parse tree browser. I was inspired by the vscode breadcrumbs bar (bbar)

![breadcrumb configuration option](serverModule.png)|
|:--:|
| <b>Breadcrumb bar on top with opened <a href="https://code.visualstudio.com/api/references/vscode-api#QuickPick">quickpick</a> menu</b>|


## Language service provider

## breadcrumb bar

This bar allows to explore and navigate 
code for the file with thee cursor focus seen as an arborescence. 
You can even do that navigation from the keyboard.
This breadcrumbs bar (bbar) is composed of 
two sections if the language service provider (LSP) is supported 
for the language for the current file.
Enries from the first section are the path from the git repo folder to 
the current file. The next entries are recursive structures that lead to 
the one under the cursor.
Currently raku does not support the LSP so we got only the first bbar section.

It is tempting to use a breadcrumb bar for navigating the raku parse tree.
It is possible to generate json from the raku parse tree to communicate with 
vscode to generate the breadcrumb bar content.



I am tempted to create a second breadcrumb bar instead of using LSP
to have more real estate. But it would oblige me to mess with vscode internals.
There is no support to add a second breadcrumb bar.
With the developper tools, I was able to find the css class breadcrumbs-tab and 
from that the code that handle breadcrumb bar. So I possibly could write an extension 
that would bypass that limitation of vscode API.

Raku parse trees are very deep. 
To save real estate, I would like subpath which matches reduce the same string to
be displayable.

raku parse is done in stages and you can display the output of
a particular state. Here we have the parse tree for the 
`say "hello"` expression. Each correspond to the reduction for a rule.
We see that the EXPR/args/arglist/value/quote is the path for the 
reduction of the `"hello"` string. 

To save space on the bbar 

## parse tree 

```
raku --target=parse -e 'say "hello"'
- statementlist: say "hello"
  - statement: 1 matches
    - EXPR: say "hello"
      - args:  "hello"
        - arglist: "hello"
          - EXPR: "hello"
            - value: "hello"
              - quote: "hello"
                - nibble: hello
      - longname: say
        - name: say
          - morename:  isa NQPArray
          - identifier: say
        - colonpair:  isa NQPArray
```
<p><center><b>Parse tree</b></center></p>



by a menu under the top one of the subpath. I don't think it is possible with the LSP
API and/or the current breadcrum API.

I could do that but supporting this second breacrumb bar would lead 
me to no progress to support full LSP services for raku.
then I discovered that the configuration 

|![breadcrumb configuration option](breadcrumbfilepath.png)|
|:--:|
| <b>An option to control file paths display in a breadcrumb bar</b>|


So far, to start simple, I want to navigate the parse tree for a unique file
with a json parse tree generated in advance.


. So getting 
rid of the first bbar section would not be a nuisance as long as I am 
interested in raku parse tree.










## Menus and commands

As I want to create a breadcrumb bar to navigate parse trees I 
must understand the code. 



An .appendMenuItem and .registerCommandAndKeybindingRule use 
the same id breacrumbs.focusAndSelect. This common id  probably explains 
why the title set in the first appears in the keybinding search panel 
instead of the id.


```typescript
MenuRegistry.appendMenuItem(MenuId.CommandPalette, {
	command: {
		id: 'breadcrumbs.focusAndSelect',
		title: { value: localize('cmd.focus', "Focus Breadcrumbs"), original: 'Focus Breadcrumbs' },
		precondition: BreadcrumbsControl.CK_BreadcrumbsVisible
	}
});

KeybindingsRegistry.registerCommandAndKeybindingRule({
	id: 'breadcrumbs.focusAndSelect',
	weight: KeybindingWeight.WorkbenchContrib,
	primary: KeyMod.CtrlCmd | KeyMod.Shift | KeyCode.Period,
	when: BreadcrumbsControl.CK_BreadcrumbsPossible,
	handler: accessor => focusAndSelectHandler(accessor, true)
});



// this commands is only enabled when breadcrumbs are
// disabled which it then enables and focuses
KeybindingsRegistry.registerCommandAndKeybindingRule({
	id: 'breadcrumbs.toggleToOn',
	weight: KeybindingWeight.WorkbenchContrib,
	primary: KeyMod.CtrlCmd | KeyMod.Shift | KeyCode.Period,
	when: ContextKeyExpr.not('config.breadcrumbs.enabled'),
	handler: async accessor => {
		const instant = accessor.get(IInstantiationService);
		const config = accessor.get(IConfigurationService);
		// check if enabled and iff not enable
		const isEnabled = BreadcrumbsConfig.IsEnabled.bindTo(config);
		if (!isEnabled.getValue()) {
			await isEnabled.updateValue(true);
			await timeout(50); // hacky - the widget might not be ready yet...
		}
		return instant.invokeFunction(focusAndSelectHandler, true);
	}
});


// focus/focus-and-select
function focusAndSelectHandler(accessor: ServicesAccessor, select: boolean): void {
	// find widget and focus/select
	const groups = accessor.get(IEditorGroupsService);
	const breadcrumbs = accessor.get(IBreadcrumbsService);
	const widget = breadcrumbs.getWidget(groups.activeGroup.id);
	if (widget) {
		const item = tail(widget.getItems());
		widget.setFocused(item);
		if (select) {
			widget.setSelection(item, BreadcrumbsControl.Payload_Pick);
		}
	}
}
```