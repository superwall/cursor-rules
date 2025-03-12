# Superwall Cursor Rules  

This repository contains custom [Cursor Rules](https://docs.cursor.com/context/rules-for-ai) designed to enhance your development experience with [Superwall](https://superwall.com/docs/using-superwall-with-cursor). By adding these rules, Cursor AI can generate better code suggestions tailored to the Superwall SDK, making integration faster and more accurate.  

Currently, this is available for iOS (Swift), with support for more SDKs and languages coming soon.  

## 🚀 Getting Started  

To set up Superwall-specific rules in Cursor:  

1. Download the User Rule file for iOS from this repo: <a id="raw-url" href="https://github.com/superwall/cursor-rules/raw/main/ios-swiftui-cursor-rules-superwall-sdk.md">`ios-swiftui-cursor-rules-superwall-sdk.md`</a>.  
2. Open Cursor and navigate to **Cursor → Settings… → Cursor Settings**.  
3. Under **User Rules**, paste the contents of the downloaded rule file.  

## 🔗 Combining with Existing Rules  

You can append these rules to any existing Cursor rules you’re using. For example:  

```markdown
[your existing rule]
[paste our Superwall rule below it]
```

For additional Cursor rules, check out [cursorrules.io](https://cursorrules.io).  

## 💡 Prompting Best Practices  

To get the best results from Cursor, explicitly mention the Superwall SDK in your prompts.  

✅ **Recommended:**  
```md
Using the Superwall SDK, register a placement called "drinkCoffee" and include a print statement saying `Coffee time` in the block.
```  

❌ **Less Effective:**  
```md
Register a placement called "drinkCoffee" and include a print statement saying `Coffee time` in the block.
```  

## 🔔 Stay Updated  

Superwall’s SDK evolves, and we’ll update these rules accordingly. **Watch** this repo for updates to ensure you always have the latest version.  
