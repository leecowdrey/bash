# BASH Arrays

```bash
arr=()           # Create empty array                     
arr=(1 2 3)      # Initialize array                      
${arr[2]}        # Retrieve third element                
${arr[@]}        # Retrieve all elements                
${!arr[@]}       # Retrieve array indices                
${#arr[@]}       # Calculate array size                  
arr[0]=3         # Overwrite 1st element                
arr+=(4)         # Append value(s)                      
str=$(ls)        # Save ls output as string              
arr=( $(ls) )    # Save ls output as array of files      
${arr[@]:s:n}    # Elements at indices n to s+n          
${str//ab/c}     # For a given string, replace ab with c
${arr[@]//ab/c}  # For each array item, replace ab with c
```
