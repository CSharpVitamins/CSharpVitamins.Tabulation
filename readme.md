# CSharpVitamins.Tabulation

This library aims to provide helpers for creating tabular data with minimal fuss. 

Easily create *tab* or *comma* separated values (via `CsvDefinition`), or padded and aligned columns of plain text (with `PlainTextTable`).

Available on [NuGet](https://www.nuget.org/packages/csharpvitamins.tabulation/). To install, run the following command in the Package Manager Console:

	PM> Install-Package CSharpVitamins.Tabulation



## Usage: `CsvDefinition`

`CsvDefinition` is designed for quick production of CSV style text - it aims to keep the *column header* and *how* to produce the value close together.

	// vars to be used inside column value funcs
	const decimal VIP_SPEND = 1000000M;
	
	// the definition  
	var fields = new CsvDefinition<MyEntity>
	{
	  { "Username",      row => string.Concat(row.Prefix, ":", row.Username) },
	  { "Email",         row => row.Email },
	  { "Status",        row => row.TotalSpend >= VIP_SPEND ? "VIP!$!" : "Pffft..., peon" },
	  { "Spend ($)",     row => row.TotalSpend == null ? "None" : row.TotalSpend.ToString("n2") },
	  { "Year of Birth", row => row.Yob > 1900 && row.Yob <= DateTime.UtcNow.Year ? row.Yob.Value.ToString() : null },
	  { "Enabled",       row => row.IsEnabled ? "Y" : "N" },
	  { "Flags",         row => string.Join(", ", new string[]
	                     {
	                       row.SomeFlag ? "some" : null,
	                       row.AnotherFlag ? "another" : null,
	                       row.YetAnotherFlag ? "even-more" : null
	                     }.Where(x => x != null))
	  },
	  { "Timestamp",     row => DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss.fff") }
	};
	
	// the data
	IEnumerable<MyEntity> rows = someDataSource.AsEnumerable<MyEntity>();
	
	// the result
	using (var writer = new StringWriter())
	{
		// perhaps write out some additional notes or headers
		// writer.WriteLine("# -- bof --");

		// write out some tab delimited data
		fields.Write(writer, rows, "\t");

		// writer.WriteLine("# -- eof --");
		return writer.ToString();
	}

At its heart, `CsvDefinition<T>` is a wrapper around `List<KeyValuePair<string, Func<T, string>>>` with a `Write` method - so, you can call `.Add` or `.Remove` to work with your base definition after it's created.

	if (!User.IsInRole("Admin"))
		fields.Remove("Spend $");


**Or have the definition created for you, from a class**


	var fields = new CsvDefinitionFactory().CreateFromModel<MyEntity>();

	... get data, create writer, etc...

	fields.Write(writer, rows, ",");

If you want more control over the production of the results, you can specify `Func<PropertyInfo, object, string>` converters for the type. 

	var factory = new CsvDefinitionFactory();

	// convert header names to lower case
	factory.NameConverter = (prop) => prop.Name.ToLowerInvariant();

	// make bools convert to Y/N
	factory.ValueConverters.Add(typeof(bool), (prop, value) => (bool)value ? "Y" : "N");

	// use the factory to create multiple definitions
	var def1 = factory.CreateFromModel<MyEntity>();

	var def2 = factory.CreateFromModel<AnotherClass>();



## Usage: `PlainTextTable`

A `PlainTextTable` allows padding of tabular data so it can be displayed easily as text. 
 
 * Works best with small to medium length strings of data
 * Does not support *multi-line text* (newlines) or *tabs* 
 * Since columns are padded using spaces, output from this class is best presented using a **fixed-wdith font**


#### Example 1: classic

	var tab = new PlainTextTable();

	// header
	tab.AddRow("Name", "Enabled?", "Title");
	
	// lines of data 
	foreach(var row in someDataSource)
 		tab.AddRow(row.Name, row.IsEnabled ? "Y" : "N", row.Title)

	tab.Align('L', 'C');

	return tab.ToString();


which might return

	Name    Enabled? Job          
	Dave       Y     Developer    
	Sarah      Y     Designer     
	Mustafa    N     Data Analysis
	

#### Example 2: fluid
 
Using the fluid style for a quick dump of data...

	return new PlainTextTable()
		.AddRow("Name", "Size", "Status") // optionally add a header
		.ImportRows(someDataSource.Select(x => new string[] { x.Name, x.Size, x.Status }))
		.SeparateBy(" | ")
		.Align('L', 'R', 'C')
		.TrimTrailingSpace()
		.ToString();

which might produce something like this

	Name               |    Size |  State
	TPS Report rev 362 |   42 MB | started
	readme.md          |    6 KB |  late
	TPS Report final   | 0 bytes |  late


Dividers can also be injected to separate headers, content and footers, e.g.

	return new PlainTextTable()
		.Align('L', 'R')
		.SeparateBy("   ")
		.AddRow("Name", "Size")
		.Divide('-')
		.ImportRows(someDataSource.Select(x => new string[] { x.Name, x.Size.ToString() }))
		.Divide('-')
		.AddRow(null, someDataSource.Sum(x => x.Size).ToString())
		.ToString();

resulting in...

	Name             Size
	---------------------
	File1.txt        5 KB
	File2.config    12 KB
	File3.exe      137 KB
	---------------------
	               154 KB


Happy coding!
