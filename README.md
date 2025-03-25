private prepareTableData(data: any, selectedBranches: string[]): void {
    console.log("Selected Branches:", selectedBranches);

    let topBranch: string = 'master';
    let gridInfo: any[] = [];

    for (const key in data) {
        if (data.hasOwnProperty(key)) {
            // Extract children properly
            const children = Object.entries(data[key]).map(([childKey, value]) => ({
                "data": { name: childKey, ...value }
            }));

            let branchSelection = selectedBranches.map(branch => branch['code']);

            console.log("Branch Selection:", branchSelection);

            if (branchSelection.includes('master')) {
                topBranch = 'master';
            } else if (branchSelection.length === 1) {
                topBranch = branchSelection[0];
            } else {
                topBranch = branchSelection.find(branch => branch !== this.develop) || 'master';
            }

            gridInfo.push({
                "data": { name: key, ...data[key][topBranch] },
                "children": children
            });
        }
    }

    this.files = gridInfo;
}
